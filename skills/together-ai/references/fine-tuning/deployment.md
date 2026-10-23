# Fine-tuned Model Deployment Reference
## Contents

- [Deployment Options](#deployment-options)
- [Training Parameters](#training-parameters)
- [Job Monitoring](#job-monitoring)
- [Continued Fine-tuning](#continued-fine-tuning)
- [Pricing](#pricing)


## Deployment Options

### Option 1: Dedicated Model Inference

Serve the fine-tune through [dedicated model inference](https://docs.together.ai/docs/dedicated-endpoints/overview)
(SDK/CLI 3.0+). A completed job is already a private model in your project (`model_object_id` /
`model_object_name` on the job). Prefer the CLI one-shot deploy; the SDK has no single-call
equivalent.

```bash
# By model_object_name or model_object_id (ml_...)
tg endpoints deploy "<MODEL_OBJECT_NAME>" --endpoint my-finetuned-endpoint
tg endpoints get "<ENDPOINT_ID>"   # poll until DEPLOYMENT_STATE_READY
```

If the model has multiple deployment profiles, pass `--config <cr_...>` (`tg models configs "<MODEL_OBJECT_ID>"`).

```python
from together import Together

client = Together()
identity = client.whoami()
if identity.type != "project_api_key":
    raise RuntimeError("TOGETHER_API_KEY must be a project API key")
project_id = identity.project.id

model = f"projects/{project_id}/models/<MODEL_OBJECT_ID>"
config = f"projects/{project_id}/configs/<CONFIG_ID>"  # tg models configs <model>

endpoint = client.endpoints.create(project_id=project_id, name="my-finetuned-endpoint")
deployment = client.endpoints.deployments.create(
    endpoint.id,
    project_id=project_id,
    name="prod",
    model=model,
    config=config,
    autoscaling={"min_replicas": 1, "max_replicas": 1},
)
client.endpoints.update(
    endpoint.id,
    project_id=project_id,
    traffic_split=[{"deployment_id": deployment.id, "weight": 1}],
)

# Infer on the dedicated base URL (no trailing /v1). Use endpoint.name, not the ml_ id.
inference = Together(base_url="https://api-inference.together.ai")
response = inference.chat.completions.create(
    model=endpoint.name,
    messages=[{"role": "user", "content": "Hello!"}],
    max_tokens=128,
)
```

- Per-minute hosting charges while replicas run (scale to 0/0 or `tg endpoints rm ep_... --force`)
- Guaranteed capacity and latency; no hard rate limits
- Supports LoRA and full fine-tuned models
- Multi-adapter serving on one deployment: see [multi-LoRA deployment](https://docs.together.ai/docs/post-training/multi-lora-deployment) for post-training adapters on supported bases; otherwise one adapter per deployment

### Option 2: Download Weights

Download and run locally or on your infrastructure.

```python
client.fine_tuning.download(
    id="ft-abc123",
    output="my-model/model.tar.zst",
)
```

```shell
# CLI: download model weights
together fine-tuning download ft-abc123

# Download to a specific directory
together fine-tuning download ft-abc123 --output_dir ./my-model

# Download a specific checkpoint step
together fine-tuning download ft-abc123 --checkpoint-step 48

# Download merged or adapter-only weights (LoRA jobs)
together fine-tuning download ft-abc123 --checkpoint-type merged
together fine-tuning download ft-abc123 --checkpoint-type adapter
```

```shell
# Extract the downloaded archive
tar -xf model-name.tar.zst
```

Options:
- `--output_dir`, `-o` -- Specify the output directory
- `--checkpoint-step`, `-s` -- Download a specific checkpoint's weights (default: latest)
- `--checkpoint-type` -- `default`, `merged`, or `adapter` (merged/adapter only for LoRA jobs)

## Training Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | Required | Base model |
| `training_file` | string | Required | File ID from upload |
| `validation_file` | string | - | Optional validation file |
| `suffix` | string | - | Custom model name suffix |
| `n_epochs` | int | 1-3 | Training epochs |
| `n_checkpoints` | int | 1 | Checkpoints to save |
| `batch_size` | int/str | `"max"` | Batch size (or "max" for auto) |
| `learning_rate` | float | ~1e-5 | Learning rate |
| `warmup_ratio` | float | 0 | Warmup step ratio |
| `lora` | bool | true | Use LoRA method |
| `lora_r` | int | 64 | LoRA rank. Per-model max -- 64 for most models; 16 for Moonshot Kimi-K2 family, Z.ai GLM-5/5.1, and DeepSeek R1/V3 (non-distill) families. See [supported-models.md](supported-models.md). |
| `lora_alpha` | int | 16 | LoRA scaling factor |
| `train_on_inputs` | bool/str | "auto" | Train on prompts/user msgs |
| `n_evals` | int | 0 | Validation evaluations (>0 to use validation set) |
| `wandb_api_key` | string | - | W&B integration |
| `from_checkpoint` | string | - | Continue from previous job ID |

### DPO-specific Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `training_method` | string | "sft" | Set to `"dpo"` for preference tuning |
| `dpo_beta` | float | 0.1 | Deviation control (0.05-0.9) |

### VLM-specific Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `train_vision` | bool | false | Update vision encoder weights |

### BYOM-specific Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `from_hf_model` | string | - | HuggingFace model ID |
| `hf_api_token` | string | - | HuggingFace token (for private repos) |

## Job Monitoring

### Status Flow
`Pending` -> `Queued` -> `Running` -> `Uploading` -> `Completed`

### Python SDK

```python
from together import Together

client = Together()

# Get status
status = client.fine_tuning.retrieve(job_id)
print(status.status)

# List events
events = client.fine_tuning.list_events(id=job_id)
for event in events.data:
    print(event.message)

# List checkpoints
checkpoints = client.fine_tuning.list_checkpoints(id=job_id)
for cp in checkpoints:
    print(f"Step {cp.step}: {cp.metrics}")

# List per-step training metrics (loss, learning rate, grad norm, eval/loss, ...)
metrics = client.fine_tuning.list_metrics(job_id)
for step in metrics.metrics:
    print(step)
```

`list_metrics` accepts optional filters for trimming long runs:

| Parameter | Type | Description |
|-----------|------|-------------|
| `global_step_from` | int | Return only metrics with `global_step` >= this value |
| `global_step_to` | int | Return only metrics with `global_step` <= this value |
| `logged_at_from` | str or datetime | Return only metrics logged at or after this ISO 8601 timestamp |
| `logged_at_to` | str or datetime | Return only metrics logged at or before this ISO 8601 timestamp |
| `resolution` | int | Cap the response at this many uniformly sampled training points (eval metrics are always returned in full) |

Each entry is either a training step (`train/global_step`, `train/loss`, `train/learning_rate`, `train/grad_norm`, ...) or an eval step (`eval/loss`, ...). When both occur at the same step, two separate objects are returned.

### CLI

```shell
together fine-tuning retrieve <JOB_ID>
together fine-tuning list-events <JOB_ID>
together fine-tuning list-checkpoints <JOB_ID>
together fine-tuning list-metrics <JOB_ID>            # ASCII charts (default)
together fine-tuning list-metrics <JOB_ID> --json     # raw JSON output
together fine-tuning list
together fine-tuning cancel <JOB_ID>
together fine-tuning delete <JOB_ID>
```

`list-metrics` also accepts `--global-step-from`, `--global-step-to`, `--logged-at-from`, `--logged-at-to`, and `--resolution` for the same filtering behavior as the Python SDK.

### cURL

```shell
# Retrieve job details
curl "https://api.together.xyz/v1/fine-tunes/ft-abc123" \
  -H "Authorization: Bearer $TOGETHER_API_KEY"

# List events
curl "https://api.together.xyz/v1/fine-tunes/ft-abc123/events" \
  -H "Authorization: Bearer $TOGETHER_API_KEY"

# List checkpoints
curl "https://api.together.xyz/v1/fine-tunes/ft-abc123/checkpoints" \
  -H "Authorization: Bearer $TOGETHER_API_KEY"

# Cancel job
curl -X POST "https://api.together.xyz/v1/fine-tunes/ft-abc123/cancel" \
  -H "Authorization: Bearer $TOGETHER_API_KEY"

# Delete job
curl -X DELETE "https://api.together.xyz/v1/fine-tunes/ft-abc123" \
  -H "Authorization: Bearer $TOGETHER_API_KEY"
```

## Continued Fine-tuning

Resume from a previous job's checkpoint:

```python
response = client.fine_tuning.create(
    training_file=new_file_id,
    model="Qwen/Qwen3-8B",
    from_checkpoint=previous_job_id,
)
```

## Pricing

- Based on total tokens processed: `total_tokens x per_token_rate`
- `total_tokens = (n_epochs x training_tokens) + (n_evals x validation_tokens)`
- Cost varies by model size, method (LoRA vs Full), and type (SFT vs DPO)
- No minimum price -- pay only for tokens processed
- Exact token count and price available after tokenization via dashboard or
  `together fine-tuning retrieve $JOB_ID`
- Dedicated endpoint hosting charges are separate (per-minute while running)
