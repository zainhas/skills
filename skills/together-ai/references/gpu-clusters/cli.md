# CLI Reference for GPU Clusters
## Contents

- [Installation](#installation)
- [Authentication](#authentication)
- [Cluster Commands](#cluster-commands)
- [Remediations](#remediations)
- [SSH](#ssh)
- [Storage Commands](#storage-commands)
- [Instance Types](#instance-types)
- [Driver Versions](#driver-versions)


The Together CLI (`tg`) manages GPU clusters, remediations, and storage. It ships with the
Together Python SDK. Use `tg clusters ...` on CLI 2.40+. CLI 3.0 removed `tg beta clusters`.
`together` is an alias of `tg`.

## Installation

```shell
uv tool install "together[cli]"
tg --help
```

Upgrade with `uv tool upgrade "together[cli]"`. For CI, prefer `uvx "together[cli]" clusters list`.

## Authentication

```shell
export TOGETHER_API_KEY=<your_key>
```

`--api-key` overrides the environment variable. Scripts should use the env var so the token does
not appear in process lists. From CLI 2.41, interactive `tg login` is available for humans.

## Global Flags

| Flag | Description |
|------|-------------|
| `--json` | Return JSON. Also suppresses interactive prompts. |
| `--non-interactive` | Disable prompts. Required in CI/CD. |
| `--api-key [string]` | Together API key. Falls back to `TOGETHER_API_KEY`. |
| `--project [string]` | Project ID (also `TOGETHER_PROJECT_ID`). Needed for preemptible GPUs. |
| `--timeout [number]` | Request timeout, in seconds. |
| `--max-retries [number]` | Maximum number of HTTP retries. |
| `--debug` | Enable debug logging. |
| `--help` | Print help for the prefixed command. |

## Cluster Commands

### `clusters create`

```shell
tg clusters create [OPTIONS]
```

Run with no flags for an interactive prompt. Pass `--non-interactive` or `--json` to skip prompts
(also automatic when no TTY is attached).

**Core options:**

| Flag | Type | Description |
|------|------|-------------|
| `--name` | string | Cluster name |
| `--num-gpus` | number | GPU count (multiple of 8) |
| `--gpu-type` | enum | `H100_SXM`, `H200_SXM`, `B200_SXM`, `B300_SXM`, `H100_SXM_INF`, `L40_PCIE`, `RTX_6000_PCI` |
| `--region` | string | Region (`clusters list-regions`) |
| `--billing-type` | enum | `ON_DEMAND`, `RESERVED`, or `SCHEDULED_CAPACITY` |
| `--duration-days` | number | Reservation length (only with `RESERVED`) |
| `--nvidia-driver-version` | string | NVIDIA driver (pair with `--cuda-version`) |
| `--cuda-version` | string | CUDA version |
| `--os` | string | OS image for the pair (default `ubuntu-22.04`) |
| `--nvidia-version-id` | string | Catalog `id` from `list-regions`; replaces the three version flags |
| `--cluster-type` | enum | `KUBERNETES` or `SLURM` |
| `--volume` | string | Existing volume ID (required in non-interactive mode when attaching storage) |
| `--capacity-pool-id` | string | Capacity pool for `SCHEDULED_CAPACITY` |
| `--num-capacity-pool-gpus` | number | GPUs drawn from the pool (multiple of 8) |
| `--num-reserved-gpus` | number | Prepaid reserved GPUs (hybrid scheduled / reserved) |
| `--reservation-start-time` | string | Scheduled window start (RFC 3339); omit to start now |
| `--reservation-end-time` | string | Scheduled window end (RFC 3339) |
| `--num-preemptible-gpus` | number | Preemptible GPUs (multiple of 8; needs a project) |
| `--non-interactive` | -- | Skip prompts |
| `--json` | -- | JSON output |

Non-interactive driver selection: either `--nvidia-version-id` alone, or
`--nvidia-driver-version` + `--cuda-version` (optionally `--os`).

```shell
tg clusters create \
  --name my-training-cluster \
  --num-gpus 8 \
  --gpu-type H100_SXM \
  --region us-central-8 \
  --cuda-version 12.6 \
  --nvidia-driver-version 560 \
  --billing-type ON_DEMAND \
  --cluster-type KUBERNETES \
  --non-interactive
```

### `clusters list` / `retrieve` / `update`

```shell
tg clusters list
tg clusters retrieve [CLUSTER_ID]
tg clusters update [CLUSTER_ID] --num-gpus 16
```

`list` pages 20 clusters in text mode; use `--after <CLUSTER_ID>` for the next page. `--json`
returns every cluster. Update can also raise `--num-reserved-gpus` / `--num-capacity-pool-gpus`
and extend `--reservation-end-time` where enabled. `--cluster-type` on update is accepted but
ignored by the API.

### `clusters delete`

```shell
tg clusters delete [CLUSTER_ID]
```

Interactive sessions confirm before deleting. Pass `--force`, or global `--non-interactive` /
`--json`, to skip the prompt.

### `clusters list-regions`

```shell
tg clusters list-regions
```

Each `driver_versions[]` entry includes `id`, `cuda_version`, `nvidia_driver_version`, and `os`.
Pass `id` to create as `--nvidia-version-id`.

### `clusters get-credentials`

```shell
tg clusters get-credentials [CLUSTER_ID] --set-default-context --no-json
```

Only project admins can download the admin kubeconfig. Useful flags: `--file`, `--context-name`,
`--overwrite-existing`, `--set-default-context`.

## Remediations

```shell
tg clusters remediations create [CLUSTER_ID] [INSTANCE_ID] \
  --mode HOST_POWER_CYCLE --reason "manual cycle"

tg clusters remediations list [CLUSTER_ID] [INSTANCE_ID]
tg clusters remediations retrieve [REMEDIATION_ID]
tg clusters remediations approve [REMEDIATION_ID] [--mode VM_ONLY]
tg clusters remediations reject [REMEDIATION_ID] --reason "not needed"
tg clusters remediations cancel [REMEDIATION_ID]
```

Modes: `VM_ONLY` (quick reprovision), `HOST_AWARE` (migrate), `EVICT_WITHOUT_REPLACEMENT`,
`REBOOT_VM`, `HOST_POWER_CYCLE` (Cycle; hardware/account gated, else `403`).
List filters: `--mode`, `--state` (`PENDING_APPROVAL`, `PENDING`, `RUNNING`, `SUCCEEDED`,
`FAILED`, `CANCELLED`, `AUTO_RESOLVED`, `QUARANTINING`, `QUARANTINED`), `--trigger`.
Omit `INSTANCE_ID` (or use `-` in the API) to list the whole cluster.

## SSH

```shell
tg clusters ssh https://dex.<LOCATION>.cloud.together.ai/t-<CLUSTER_ID> \
  --login <LOGIN> [--host <HOST>]
```

OIDC SSH needs CLI 2.40+ and Python 3.10+. `<LOCATION>` is the datacenter segment of node
hostnames (for example `s2.ca-west-1b`), not the region name. Key-based jump host form:

```shell
ssh -J <LOGIN>@ssh.t-<CLUSTER_ID>.<LOCATION>.cloud.together.ai <LOGIN>@<HOST>
```

## Storage Commands

```shell
tg clusters storage create --volume-name my-data --size-tib 2 --region us-central-8
tg clusters storage list
tg clusters storage retrieve [VOLUME_ID]
tg clusters storage delete [VOLUME_ID] --force
```

Delete confirms interactively unless `--force`, `--non-interactive`, or `--json` is set. The
volume must not be attached to any cluster.

## Instance Types

| CLI Value | GPU | Memory | Notes |
|-----------|-----|--------|-------|
| `H100_SXM` | NVIDIA H100 | 80GB | InfiniBand networking |
| `H100_SXM_INF` | NVIDIA H100 | 80GB | Inference-optimized, lower IB bandwidth |
| `H200_SXM` | NVIDIA H200 | 141GB | InfiniBand networking |
| `B200_SXM` | NVIDIA B200 | 192GB | InfiniBand networking |
| `B300_SXM` | NVIDIA B300 | — | Availability varies by region |
| `L40_PCIE` | NVIDIA L40 | 48GB | PCIe |
| `RTX_6000_PCI` | NVIDIA RTX 6000 | 24GB | PCIe |

## Driver Versions

Prefer the live `list-regions` pairs (and their catalog `id`). Combined `driver_version` strings
still follow `CUDA_{major}_{minor}_{nvidia}` (for example `CUDA_12_6_560`).
