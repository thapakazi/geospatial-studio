# Fine-Tuning Locally on an Apple GPU (MPS)

A local deployment on an Apple-silicon Mac has no NVIDIA GPU, so the studio's in-cluster
fine-tuning jobs fall back to CPU. On a 300M-parameter backbone that is too slow to be
useful — and because the container images are `linux/amd64`, the job also runs under
emulation, which disables optimised CPU kernels entirely:

```
GPU available: False, used: False
NNPACK could not initialize! Reason: Unsupported hardware.
```

The workflow below instead uses the studio to **produce** the training configuration, runs
TerraTorch **natively on the Mac's GPU** through PyTorch's Metal (MPS) backend, and then
**registers the trained model back** into the studio so it can be used for inference.

!!! info "What this is for"
    Local training is for development and small experiments. For production training, deploy
    to a cluster with NVIDIA GPUs, where the normal one-step `POST /v2/submit-tune` flow
    works end to end.

---

## Prerequisites

### A separate Python environment

TerraTorch requires `torchgeo >= 0.9`, which requires **Python 3.12 or newer**. The studio's
deployment scripts target **Python 3.11**. These cannot share one virtual environment — create
a second one:

```bash
python3.12 -m venv .venv-terratorch        # 3.12, 3.13 or 3.14
./.venv-terratorch/bin/pip install terratorch
```

### MLflow

The configuration the studio generates always includes an `MLFlowLogger`, but `mlflow` is not
a TerraTorch dependency. Without it, training aborts immediately with
`ModuleNotFoundError: Requirement 'mlflow>=1.0.0' not met`:

```bash
./.venv-terratorch/bin/pip install "mlflow>=1.0.0"
```

### Confirm the GPU is usable

```bash
./.venv-terratorch/bin/python -c "import torch; print(torch.backends.mps.is_available())"
# True
```

---

## 1. Onboard the artefacts

Onboard a dataset, a backbone and a tuning template as usual — these steps are identical to a
cluster deployment:

```bash
export STUDIO_API_KEY="<your api key>"
export UI_ROUTE_URL="https://localhost:4180"

python populate-studio/populate-studio.py datasets    # e.g. Wildfire burn scars
python populate-studio/populate-studio.py backbones   # e.g. Prithvi_EO_V2_300M
python populate-studio/populate-studio.py templates   # e.g. Segmentation - Generic
```

Wait for the dataset to reach `Succeeded` before continuing.

---

## 2. Generate the training configuration

The dry-run endpoint returns a complete TerraTorch configuration without submitting a job:

```bash
payload='{
  "name": "burn-scars-demo",
  "description": "Segmentation",
  "dataset_id": "<dataset id>",
  "base_model_id": "<backbone id>",
  "tune_template_id": "<template id>",
  "model_parameters": {
    "runner":    { "max_epochs": "1" },
    "optimizer": { "lr": 6e-05, "type": "AdamW" }
  }
}'

echo $payload | curl -X POST "${UI_ROUTE_URL}/studio-gateway/v2/submit-tune/dry-run" \
  --header 'Content-Type: application/json' \
  --header "X-API-Key: $STUDIO_API_KEY" \
  --insecure --data @- > config.yaml
```

---

## 3. Make the data reachable locally

```bash
./deployment-scripts/localize_config.sh config.yaml
```

This rewrites the MLflow URL and the in-cluster paths to `~/studio-data/…`.

!!! warning "The rewritten data path may be empty"
    `localize_config.sh` assumes the fine-tuning PVCs are backed by a host directory. When the
    deployment uses the S3 CSI driver (`cos-s3-csi-s3fs-sc`, the default for Lima and kind),
    the dataset lives **in a bucket**, and `~/studio-data/ft-data-pvc` does not exist. The
    rewrite then points at nothing and training fails on missing files.

    Download the dataset out of the object store first. The layout under
    `~/studio-data/ft-data-pvc/<dataset-id>/` must mirror the bucket:
    `training_data/<modality>/`, `labels/` and `split_files/`.

    A full dataset can be large — the burn-scars example is ~13 GB — so for a first run
    consider copying a subset and writing reduced `split_files` to match.

---

## 4. Point the configuration at the Apple GPU

Three further edits are needed, which `localize_config.sh` does not make:

```yaml
trainer:
  accelerator: mps          # was: auto
  precision: 32-true        # was: 16-mixed - MPS has no fp16/bf16 AMP path
data:
  init_args:
    num_workers: 0          # avoids multiprocessing issues with MPS on macOS
```

---

## 5. Train

```bash
export PYTORCH_ENABLE_MPS_FALLBACK=1
./.venv-terratorch/bin/terratorch fit -c config.yaml
```

`PYTORCH_ENABLE_MPS_FALLBACK` lets any operation Metal does not implement fall back to the
CPU instead of raising. Confirm the GPU is in use:

```
GPU available: True (mps), used: True
Trainable params: 324 M
```

Metrics are reported to the same MLflow instance as the cluster, so the run appears at
<http://localhost:5000> alongside in-cluster tunes. The checkpoint is written to the
`ModelCheckpoint` `dirpath` from the configuration.

---

## 6. Register the trained model back into the studio

`POST /v2/upload-completed-tunes` takes **URLs**, not file uploads, and the gateway fetches
them from inside the cluster. Upload both artefacts to a bucket the studio can reach, then
pre-sign them using the **in-cluster** endpoint — signing is offline, so the client used to
generate the URL does not need to connect:

```python
import ibm_boto3
from ibm_botocore.client import Config
cfg = Config(signature_version="s3v4", s3={"addressing_style": "path"})

# upload via the port-forwarded endpoint
up = ibm_boto3.client("s3", endpoint_url="https://127.0.0.1:9000",
                      aws_access_key_id=KEY, aws_secret_access_key=SECRET,
                      config=cfg, verify=False)
up.upload_file("config.yaml",  BUCKET, "demo/config.yaml")
up.upload_file(CHECKPOINT,     BUCKET, "demo/model.ckpt")

# pre-sign with the in-cluster hostname so the gateway can fetch it
sign = ibm_boto3.client("s3", endpoint_url="https://minio.default.svc.cluster.local:9000",
                        aws_access_key_id=KEY, aws_secret_access_key=SECRET,
                        config=cfg, verify=False)
cfg_url  = sign.generate_presigned_url("get_object",
             Params={"Bucket": BUCKET, "Key": "demo/config.yaml"}, ExpiresIn=86400)
ckpt_url = sign.generate_presigned_url("get_object",
             Params={"Bucket": BUCKET, "Key": "demo/model.ckpt"},  ExpiresIn=86400)
```

!!! tip "Upload the original configuration, not the MPS one"
    Register the configuration produced in step 2, before the step-4 edits. The model
    architecture is identical, but a configuration carrying `accelerator: mps` is wrong for
    in-cluster inference.

Then register the tune:

```bash
payload='{
  "name": "burn-scars-mps-demo",
  "description": "Fine-tuned locally on Apple GPU",
  "tune_config_url": "<cfg_url>",
  "tune_checkpoint_url": "<ckpt_url>"
}'

echo $payload | curl -X POST "${UI_ROUTE_URL}/studio-gateway/v2/upload-completed-tunes" \
  --header 'Content-Type: application/json' \
  --header "X-API-Key: $STUDIO_API_KEY" \
  --insecure --data @-
# {"message":"Upload started","tune_id":"geotune-..."}
```

The download runs asynchronously in the Celery worker. Poll until the status reaches
`Finished`:

```bash
curl -sk -H "X-API-Key: $STUDIO_API_KEY" \
  "${UI_ROUTE_URL}/studio-gateway/v2/tunes/<tune_id>"
```

!!! warning "Self-signed certificates will stall this step"
    If the tune stays `Pending` and the Celery worker logs `CERTIFICATE_VERIFY_FAILED`, the
    worker does not trust the object store's self-signed certificate. See
    [Troubleshooting](../../deployment/troubleshooting.md#uploading-a-completed-tune-hangs-in-pending).

---

## 7. Run inference with the tuned model

Once the tune is `Finished`, use it over a spatial and temporal domain:

```bash
payload='{
  "model_display_name": "burn-scars-mps-demo",
  "location": "Dakhin Petbaha, Assam, India",
  "description": "Burn scar inference",
  "spatial_domain": { "bbox": [[92.703396, 26.247896, 92.748087, 26.267903]],
                      "urls": [], "tiles": [], "polygons": [] },
  "temporal_domain": ["2024-07-25_2024-07-28"]
}'

echo $payload | curl -X POST "${UI_ROUTE_URL}/studio-gateway/v2/tunes/<tune_id>/try-out" \
  --header 'Content-Type: application/json' \
  --header "X-API-Key: $STUDIO_API_KEY" \
  --insecure --data @-
```

Progress is visible on the inferences page in the UI, and the outputs are written to the
object store.
