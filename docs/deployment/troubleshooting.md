# Deployment Troubleshooting

Known issues that can block a deployment, with their symptoms and fixes. Each entry lists
the error you will actually see in the logs, so you can match on that rather than guess.

---

## MinIO image is no longer publicly available

!!! failure "Symptom"
    The `minio` pod never starts and the deploy script loops on
    `kubectl wait --for=condition=ready pod -l app=minio` until it exhausts its retries.

    ```
    Failed to pull image "quay.io/minio/minio:latest":
      failed to resolve reference "quay.io/minio/minio:latest":
      unexpected status from HEAD request to
      https://quay.io/v2/minio/minio/manifests/latest: 401 UNAUTHORIZED
    ```

**Cause.** MinIO withdrew its community container images from public registries. The
repository is unavailable anonymously everywhere it was previously published:

| Source | Result |
|---|---|
| `quay.io/minio/minio` | `401 UNAUTHORIZED` |
| Docker Hub `minio/minio` | repository not found |
| `mirror.gcr.io`, `public.ecr.aws`, `ghcr.io` | no mirror |

This affects every deployment target — OpenShift, Kubernetes, kind and Lima alike.

**Fix.** Substitute an S3-compatible object store. `deployment-scripts/minio-deployment.yaml`
and `deployment-scripts/images-pre-puller/image-prepuller.yaml` are the only two places the
image is referenced.

Any replacement must satisfy what the rest of the deployment assumes:

- **TLS on port 9000** — `deployment-scripts/template/cos-s3-csi-sc.yaml` hardcodes
  `https://minio.<namespace>.svc.cluster.local:9000`
- **Path-style S3** addressing, since `create_buckets.py` connects with `verify=False`
- **Presigned URLs**, used by the gateway's `/v2/file-share` endpoint
- **s3fs compatibility**, because `ibm-object-csi-driver` mounts buckets as PVCs

!!! tip "Keep the Kubernetes object names unchanged"
    Leave the Deployment named `minio` with its `app: minio` labels and the Service name
    `minio`. The in-cluster DNS name, the deploy script's `kubectl wait -l app=minio`, and
    the CSI storage class all depend on them. Only the pod internals need to change.

### Example: RustFS

[RustFS](https://github.com/rustfs/rustfs) is Apache-2.0 licensed and publishes multi-arch
images (`linux/amd64` and `linux/arm64`). Its container already exposes ports 9000/9001 and
uses `/data`, matching the existing manifest. Four things differ from MinIO:

| | MinIO | RustFS |
|---|---|---|
| Startup | `args: [server, /data, --console-address, ":9001"]` | own entrypoint — **remove `args`** |
| User | root | **UID 10001** — requires `fsGroup` |
| TLS certs | `/root/.minio/certs` | `RUSTFS_TLS_PATH`, named `rustfs_cert.pem` / `rustfs_key.pem` |
| Credentials | `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` | `RUSTFS_ACCESS_KEY` / `RUSTFS_SECRET_KEY` |

```yaml
    spec:
      securityContext:
        fsGroup: 10001
      containers:
      - name: minio
        image: rustfs/rustfs:latest
        env:
        - name: RUSTFS_ACCESS_KEY
          value: "minioadmin"
        - name: RUSTFS_SECRET_KEY
          value: "minioadmin"
        - name: RUSTFS_VOLUMES
          value: "/data"
        - name: RUSTFS_ADDRESS
          value: ":9000"
        - name: RUSTFS_CONSOLE_ENABLE
          value: "true"
        - name: RUSTFS_CONSOLE_ADDRESS
          value: ":9001"
        - name: RUSTFS_TLS_PATH
          value: "/certs"
        volumeMounts:
        - name: minio-tls-certs
          mountPath: /certs
          readOnly: true
        - name: data
          mountPath: /data
      volumes:
      - name: minio-tls-certs
        secret:
          secretName: minio-tls-secret
          items:
          - key: tls.crt
            path: rustfs_cert.pem
          - key: tls.key
            path: rustfs_key.pem
```

The existing `minio-tls-secret` needs no change — it already renames keys to filenames, so
only the target names differ.

---

## Keycloak is OOMKilled during startup

!!! failure "Symptom"
    `keycloak` enters `CrashLoopBackOff`. The startup probe fails with
    `connect: connection refused` on port 9000, and the container's last state is:

    ```
    reason=OOMKilled  exit=137
    ```

    The logs stop after `Updating the configuration and installing your custom providers,
    please wait.` and never reach the Quarkus startup banner.

**Cause.** Keycloak performs a Quarkus build (augmentation) step on first start, which needs
more memory than the steady-state server. `deployment-scripts/keycloak-deployment.yaml` caps
the container at `1Gi`, which the build step exceeds.

**Fix.** Raise the limit. 3Gi is comfortable; the server settles well below it afterwards.

```yaml
        resources:
          requests:
            cpu: 250m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 3Gi
```

---

## MLflow crashes with `No module named 'psycopg'`

!!! failure "Symptom"
    `geofm-mlflow` enters `CrashLoopBackOff` with:

    ```
    File ".../sqlalchemy/dialects/postgresql/psycopg.py", line 497, in import_dbapi
        import psycopg
    ModuleNotFoundError: No module named 'psycopg'
    ```

**Cause.** A driver mismatch introduced by an upstream dependency change, not by
configuration. The MLflow deployment installs **psycopg2**:

```bash
pip install --user psycopg2-binary
```

but the backend URI is built with a bare `postgresql://` scheme in
`geospatial-studio/templates/_helpers.tpl`. SQLAlchemy 2.1 changed the default PostgreSQL
driver for bare `postgresql://` URIs from `psycopg2` to `psycopg` (v3). Because
`ghcr.io/mlflow/mlflow:latest` now ships SQLAlchemy 2.1, the URI resolves to a driver that
is never installed.

You can confirm the resolution in any MLflow image:

```python
>>> from sqlalchemy.engine.url import make_url
>>> make_url("postgresql://u:p@h:5432/db").get_dialect().__module__
'sqlalchemy.dialects.postgresql.psycopg'       # v3 — not installed
```

**Fix.** Name the driver explicitly instead of relying on the default, so the URI keeps
matching the package that is installed:

```diff
- postgresql://{{ .Values.global.postgres.postgres_user }}:...
+ postgresql+psycopg2://{{ .Values.global.postgres.postgres_user }}:...
```

!!! note
    Pinning the scheme is preferable to installing psycopg v3, because the explicit form is
    stable across future SQLAlchemy default changes.

---

## Dataset onboarding fails with an invalid `imagePullPolicy`

!!! failure "Symptom"
    `POST /v2/datasets/onboard` returns `500 - {"detail":"b''"}` and the gateway logs show:

    ```
    The Job "onboarding-v2-pipeline-<id>" is invalid:
      spec.template.spec.containers[0].imagePullPolicy:
      Unsupported value: "IMAGE_PULL_POLICY":
      supported values: "Always", "IfNotPresent", "Never"
    ```

**Cause.** A placeholder-token mismatch in `geospatial-studio-core`. The job template contains
a bare token, but the substitution step replaces a differently-named one:

```yaml
# gfmstudio/fine_tuning/deployment/k8-dataset-onboarding-v2-pipeline.tpl.yaml
imagePullPolicy: IMAGE_PULL_POLICY
```

```python
# gfmstudio/fine_tuning/api.py  (replace_dataset_id_command)
f"s|FTUNING_IMAGE_PULL_POLICY|{settings.IMAGE_PULL_POLICY}|g; "
```

Nothing rewrites the bare `IMAGE_PULL_POLICY`, so the literal string reaches the API server
and the Job is rejected. The sibling fine-tuning template uses `${IMAGE_PULL_POLICY}` and is
unaffected, which is why only dataset onboarding breaks.

**Fix.** Align the template token with the substitution:

```diff
- imagePullPolicy: IMAGE_PULL_POLICY
+ imagePullPolicy: FTUNING_IMAGE_PULL_POLICY
```

!!! note "Workaround without rebuilding the image"
    The template ships inside the gateway image and its directory is not writable. You can
    override just that file with a ConfigMap mounted via `subPath` at
    `/app/gfmstudio/fine_tuning/deployment/k8-dataset-onboarding-v2-pipeline.tpl.yaml`.

---

## Uploading a completed tune hangs in `Pending`

!!! failure "Symptom"
    `POST /v2/upload-completed-tunes` returns `201`, but the tune stays `Pending`. The Celery
    worker retries forever:

    ```
    SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED]
      certificate verify failed: self-signed certificate'))
    ConnectionError: Max retries reached in dowload_with_backoff
    Task invoke_tune_download retry: Retry in 60s
    ```

**Cause.** `tune_config_url` and `tune_checkpoint_url` are fetched by the Celery worker with
certificate verification enabled. When those URLs point at the studio's own object store, the
download fails, because the deployment generates a **self-signed** certificate for it:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -subj "/CN=$OC_PROJECT.svc.cluster.local"
```

The worker has no `REQUESTS_CA_BUNDLE` set, and the `*-trusted-certificates-ca` ConfigMap is
empty and not mounted into it — so there is no path by which that certificate is trusted.

**Fix.** Give the worker a CA bundle containing the object store's certificate. Append it to
the existing `certifi` bundle rather than replacing it, or other HTTPS calls (model downloads
from Hugging Face, data connectors) will start failing instead:

```bash
kubectl exec <celery-pod> -- sh -c 'cat $(python -c "import certifi;print(certifi.where())")' > ca-base.pem
kubectl get secret minio-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d > store.crt
cat ca-base.pem store.crt > ca-bundle.pem
kubectl create configmap studio-ca-bundle --from-file=ca-bundle.pem
```

Mount it and point the worker at it with `REQUESTS_CA_BUNDLE` and `SSL_CERT_FILE`.

!!! warning "Use `kubectl create`, not `kubectl apply`"
    A full CA bundle exceeds the 262144-byte limit for the `last-applied-configuration`
    annotation that `apply` writes, and the request is rejected.

!!! note
    Celery retries are held in worker memory. Restarting the worker to apply this fix drops
    the in-flight retry, so the upload must be re-submitted afterwards.

---

## Deploy script debug output fails on macOS

!!! warning "Symptom"
    Retry blocks print `grep` usage errors instead of diagnostics:

    ```
    grep: invalid option -- P
    usage: grep [-abcdDEFGHhIiJLlMmnOopqRSsUVvwXxZz] ...
    ```

**Cause.** `common_functions.sh` uses `grep -P` (PCRE) to extract the namespace, label
selector and pod name for its debug output. `-P` is a GNU extension; the BSD `grep` shipped
with macOS does not support it.

**Impact.** Cosmetic — only the "Debugging Information" block is lost. The deployment itself
is unaffected.

**Fix.** Use a POSIX-compatible expression, for example `grep -E` with a capture via `sed`,
or `awk`. Alternatively document `ggrep` (GNU coreutils) as a macOS prerequisite.

---

## s3fs volumes fail to mount after a cluster or VM restart

!!! failure "Symptom"
    After restarting a local VM or rebooting a node, pods with bucket-backed PVCs hang in
    `ContainerCreating` or `PodInitializing`, and the CSI driver logs:

    ```
    MountVolume.SetUp failed for volume "pvc-..." :
      rpc error: code = Internal desc = exit status 1
    ...
    Can not validate target mount point:
      /var/lib/kubelet/pods/<uid>/volumes/kubernetes.io~csi/<pv>/mount  is not a mountpoint
    ```

**Cause.** s3fs mounts are FUSE mounts and do not survive a restart. The kubelet still
references the old pod directories, and `NodePublishVolume` fails validating a target that
is no longer a mountpoint.

**Fix.** Restart the CSI node driver so it re-publishes cleanly, then recreate the affected
pods:

```bash
kubectl delete pod -n kube-system -l app=cos-s3-csi-driver
kubectl delete pod --field-selector status.phase=Unknown
```

Pods orphaned by the restart may need `--force --grace-period=0`, since the node no longer
tracks them.

---

## Pipeline pods show `0/1` but are working normally

!!! info "Symptom"
    `pipelines-*` pods report `Running` with `0/1` ready, while their logs show healthy
    polling:

    ```
    orchestrate_wrapper.py:590 - ------ No tasks found, waiting 10 seconds ---------
    ```

**Cause.** The readiness probe matches the worker's command line with
`pgrep -f "python <component>.py"`. The probe can fail to match while the process is still
starting, or where the command line differs from the pattern.

**Impact.** None functionally — the worker polls the orchestration database and picks up
tasks regardless. The pods normally report ready once fully started.
