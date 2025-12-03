# Configuration

This document outlines the environment variables available for configuring the `worker-comfyui`.

## General Configuration

| Environment Variable | Description                                                                                                                                                                                                                  | Default |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `REFRESH_WORKER`     | When `true`, the worker pod will stop after each completed job to ensure a clean state for the next job. See the [RunPod documentation](https://docs.runpod.io/docs/handler-additional-controls#refresh-worker) for details. | `false` |
| `SERVE_API_LOCALLY`  | When `true`, enables a local HTTP server simulating the RunPod environment for development and testing. See the [Development Guide](development.md#local-api) for more details.                                              | `false` |
| `COMFY_ORG_API_KEY`  | Comfy.org API key to enable ComfyUI API Nodes. If set, it is sent with each workflow; clients can override per request via `input.api_key_comfy_org`.                                                                        | –       |

## Logging Configuration

| Environment Variable | Description                                                                                                                                                      | Default |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `COMFY_LOG_LEVEL`    | Controls ComfyUI's internal logging verbosity. Options: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`. Use `DEBUG` for troubleshooting, `INFO` for production. | `DEBUG` |

## Debugging Configuration

| Environment Variable           | Description                                                                                                            | Default |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------- | ------- |
| `WEBSOCKET_RECONNECT_ATTEMPTS` | Number of websocket reconnection attempts when connection drops during job execution.                                  | `5`     |
| `WEBSOCKET_RECONNECT_DELAY_S`  | Delay in seconds between websocket reconnection attempts.                                                              | `3`     |
| `WEBSOCKET_TRACE`              | Enable low-level websocket frame tracing for protocol debugging. Set to `true` only when diagnosing connection issues. | `false` |
| `NETWORK_VOLUME_DEBUG`         | Enable detailed network volume diagnostics in worker logs. Useful for debugging model path issues. See [Network Volume Configuration](#network-volume-configuration) below. | `false` |

> [!TIP]
> **For troubleshooting:** Set `COMFY_LOG_LEVEL=DEBUG` to get detailed logs when ComfyUI crashes or behaves unexpectedly. This helps identify the exact point of failure in your workflows.

## Network Volume Configuration

When using a RunPod network volume to store your models, the worker expects a specific directory structure. If ComfyUI is not finding your models, enable diagnostics by setting `NETWORK_VOLUME_DEBUG=true`.

### Expected Directory Structure

Models must be placed in the following structure on your network volume:

```
/runpod-volume/
└── models/
    ├── checkpoints/      # Stable Diffusion checkpoints (.safetensors, .ckpt)
    ├── loras/            # LoRA files (.safetensors, .pt)
    ├── vae/              # VAE models (.safetensors, .pt)
    ├── clip/             # CLIP models (.safetensors, .pt)
    ├── clip_vision/      # CLIP Vision models
    ├── controlnet/       # ControlNet models (.safetensors, .pt)
    ├── embeddings/       # Textual inversion embeddings (.safetensors, .pt)
    ├── upscale_models/   # Upscaling models (.safetensors, .pt)
    ├── unet/             # UNet models
    └── configs/          # Model configs (.yaml, .json)
```

### Supported File Extensions

ComfyUI only recognizes files with specific extensions:

| Model Type       | Supported Extensions                |
| ---------------- | ----------------------------------- |
| Checkpoints      | `.safetensors`, `.ckpt`, `.pt`, `.pth`, `.bin` |
| LoRAs            | `.safetensors`, `.pt`               |
| VAE              | `.safetensors`, `.pt`, `.bin`       |
| CLIP             | `.safetensors`, `.pt`, `.bin`       |
| ControlNet       | `.safetensors`, `.pt`, `.pth`, `.bin` |
| Embeddings       | `.safetensors`, `.pt`, `.bin`       |
| Upscale Models   | `.safetensors`, `.pt`, `.pth`       |

> [!WARNING]
> **Common Issues:**
> - Models placed directly in `/runpod-volume/checkpoints/` instead of `/runpod-volume/models/checkpoints/` will not be found.
> - Files with incorrect extensions (e.g., `.txt`, `.zip`) will be ignored.
> - Empty directories or missing subdirectories are fine—only create the folders you need.

### Debugging Network Volume Issues

1. **Enable diagnostics** by adding `NETWORK_VOLUME_DEBUG=true` to your endpoint's environment variables.

2. **Send a test request** to your endpoint (any request will trigger the diagnostics).

3. **Check the worker logs** in the RunPod console. You'll see detailed output like:

```
======================================================================
NETWORK VOLUME DIAGNOSTICS (NETWORK_VOLUME_DEBUG=true)
======================================================================

[1] Checking extra_model_paths.yaml configuration...
    ✓ FOUND: /comfyui/extra_model_paths.yaml

[2] Checking network volume mount at /runpod-volume...
    ✓ MOUNTED: /runpod-volume

[3] Checking directory structure...
    ✓ FOUND: /runpod-volume/models

[4] Scanning model directories...

    checkpoints/:
      - my-model.safetensors (6.5 GB)

    loras/:
      - style-lora.safetensors (144.2 MB)

[5] Summary
    ✓ Models found on network volume!
======================================================================
```

4. **Disable diagnostics** once your issue is resolved by removing the environment variable or setting it to `false`.

## AWS S3 Upload Configuration

Configure these variables **only** if you want the worker to upload generated images directly to an AWS S3 bucket. If these are not set, images will be returned as base64-encoded strings in the API response.

- **Prerequisites:**
  - An AWS S3 bucket in your desired region.
  - An AWS IAM user with programmatic access (Access Key ID and Secret Access Key).
  - Permissions attached to the IAM user allowing `s3:PutObject` (and potentially `s3:PutObjectAcl` if you need specific ACLs) on the target bucket.

| Environment Variable       | Description                                                                                                                             | Example                                                    |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `BUCKET_ENDPOINT_URL`      | The full endpoint URL of your S3 bucket. **Must be set to enable S3 upload.**                                                           | `https://<your-bucket-name>.s3.<aws-region>.amazonaws.com` |
| `BUCKET_ACCESS_KEY_ID`     | Your AWS access key ID associated with the IAM user that has write permissions to the bucket. Required if `BUCKET_ENDPOINT_URL` is set. | `AKIAIOSFODNN7EXAMPLE`                                     |
| `BUCKET_SECRET_ACCESS_KEY` | Your AWS secret access key associated with the IAM user. Required if `BUCKET_ENDPOINT_URL` is set.                                      | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`                 |

**Note:** Upload uses the `runpod` Python library helper `rp_upload.upload_image`, which handles creating a unique path within the bucket based on the `job_id`.

### Example S3 Response

If the S3 environment variables (`BUCKET_ENDPOINT_URL`, `BUCKET_ACCESS_KEY_ID`, `BUCKET_SECRET_ACCESS_KEY`) are correctly configured, a successful job response will look similar to this:

```json
{
  "id": "sync-uuid-string",
  "status": "COMPLETED",
  "output": {
    "images": [
      {
        "filename": "ComfyUI_00001_.png",
        "type": "s3_url",
        "data": "https://your-bucket-name.s3.your-region.amazonaws.com/sync-uuid-string/ComfyUI_00001_.png"
      }
      // Additional images generated by the workflow would appear here
    ]
    // The "errors" key might be present here if non-fatal issues occurred
  },
  "delayTime": 123,
  "executionTime": 4567
}
```

The `data` field contains the presigned URL to the uploaded image file in your S3 bucket. The path usually includes the job ID.
