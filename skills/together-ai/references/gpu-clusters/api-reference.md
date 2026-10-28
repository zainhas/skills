# GPU Clusters API Reference
## Contents

- [Cluster Endpoints](#cluster-endpoints)
- [Storage Endpoints](#storage-endpoints)
- [Create Cluster](#create-cluster)
- [List Clusters](#list-clusters)
- [Get Cluster](#get-cluster)
- [Update / Scale Cluster](#update-scale-cluster)
- [Delete Cluster](#delete-cluster)
- [List Regions](#list-regions)
- [Create Shared Volume](#create-shared-volume)
- [List Shared Volumes](#list-shared-volumes)
- [Get Shared Volume](#get-shared-volume)
- [Update (Resize) Shared Volume](#update-shared-volume)
- [Delete Shared Volume](#delete-shared-volume)
- [Instance Types](#instance-types)
- [Driver Versions](#driver-versions)
- [Cluster Statuses](#cluster-statuses)
- [Volume Statuses](#volume-statuses)
- [Cluster Response Object](#cluster-response-object)


Base URL: `https://api.together.xyz/v1`

## Cluster Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST /compute/clusters` | Create cluster | Provision a new GPU cluster |
| `GET /compute/clusters` | List clusters | List all GPU clusters |
| `GET /compute/clusters/{id}` | Get cluster | Get cluster details |
| `PUT /compute/clusters/{id}` | Update cluster | Scale or change cluster type |
| `DELETE /compute/clusters/{id}` | Delete cluster | Remove a cluster |
| `GET /compute/regions` | List regions | Available regions, GPUs, drivers |
| `GET/POST /compute/clusters/{id}/instances/{iid}/remediations` | Remediations | List/create node remediations (`iid` may be `-` to list the cluster) |
| `GET /compute/clusters/remediations/{id}` | Get remediation | Retrieve one remediation |
| `POST .../remediations/{id}/approve|reject|cancel` | Remediation actions | Approve (optional mode override), reject, or cancel |

Remediation modes: `VM_ONLY`, `HOST_AWARE`, `EVICT_WITHOUT_REPLACEMENT`, `REBOOT_VM`,
`HOST_POWER_CYCLE`. States include `PENDING_APPROVAL`, `QUARANTINING`, `QUARANTINED`, and the
usual pending/running/terminal set. `HOST_POWER_CYCLE` returns `403` when not enabled.

## Storage Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST /compute/clusters/storage/volumes` | Create volume | Create shared storage |
| `GET /compute/clusters/storage/volumes` | List volumes | List all volumes |
| `GET /compute/clusters/storage/volumes/{id}` | Get volume | Get volume details |
| `PUT /compute/clusters/storage/volumes` | Update volume | Resize a volume |
| `DELETE /compute/clusters/storage/volumes/{id}` | Delete volume | Remove a volume |

## Create Cluster

Pass `cuda_version` and `nvidia_driver_version` (optionally `os`; default `ubuntu-22.04`).
Current SDKs accept them as first-class fields on `client.clusters.create`. Older Python
clients may still need `extra_body`. A combined `driver_version` string is optional.

```python
from together import Together
client = Together()

cluster = client.clusters.create(
    cluster_name="my-gpu-cluster",
    region="us-central-8",
    gpu_type="H100_SXM",
    num_gpus=8,
    cuda_version="12.6",
    nvidia_driver_version="560",
    billing_type="ON_DEMAND",
    cluster_type="KUBERNETES",
    # volume_id="existing-volume-id",  # optional: attach existing volume
)
print(cluster.cluster_id)
```

```typescript
import Together from "together-ai";
const client = new Together();

const cluster = await client.clusters.create({
  cluster_name: "my-gpu-cluster",
  region: "us-central-8",
  gpu_type: "H100_SXM",
  num_gpus: 8,
  cuda_version: "12.6",
  nvidia_driver_version: "560",
  billing_type: "ON_DEMAND",
  cluster_type: "KUBERNETES",
});
console.log(cluster.cluster_id);
```

```shell
curl -X POST \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "cluster_name": "my-gpu-cluster",
    "region": "us-central-8",
    "gpu_type": "H100_SXM",
    "num_gpus": 8,
    "driver_version": "CUDA_12_6_560",
    "cuda_version": "12.6",
    "nvidia_driver_version": "560",
    "billing_type": "ON_DEMAND",
    "cluster_type": "KUBERNETES"
  }' \
  https://api.together.xyz/v1/compute/clusters
```

```shell
tg clusters create \
  --name my-gpu-cluster \
  --num-gpus 8 \
  --gpu-type H100_SXM \
  --region us-central-8 \
  --cuda-version 12.6 \
  --nvidia-driver-version 560 \
  --billing-type ON_DEMAND \
  --cluster-type KUBERNETES \
  --non-interactive
```

### Create Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `cluster_name` | string | Yes | Name of the cluster |
| `region` | string | Yes | Region (use `list_regions()` / `listRegions()`) |
| `gpu_type` | string | Yes | GPU type (see Instance Types below) |
| `num_gpus` | integer | Yes | Number of GPUs (must be a multiple of 8) |
| `cuda_version` | string | Yes* | CUDA version, e.g. `"12.6"` (*or pass nvidia version catalog `id`) |
| `nvidia_driver_version` | string | Yes* | NVIDIA driver version, e.g. `"560"` |
| `os` | string | No | OS image for the pair; default `ubuntu-22.04` |
| `driver_version` | string | No | Combined string, e.g. `CUDA_12_6_560` (optional alongside split fields) |
| `billing_type` | string | Yes | `ON_DEMAND`, `RESERVED`, or `SCHEDULED_CAPACITY` |
| `cluster_type` | string | No | `KUBERNETES` (default) or `SLURM` |
| `duration_days` | integer | No | Reservation length in days (only with `RESERVED`; 3–180) |
| `capacity_pool_id` | string | No | Capacity pool for `SCHEDULED_CAPACITY` |
| `num_capacity_pool_gpus` | integer | No | GPUs from the pool (multiple of 8) |
| `num_reserved_gpus` | integer | No | Prepaid reserved GPUs (hybrid scheduled / reserved) |
| `reservation_start_time` | string | No | Scheduled window start (RFC 3339); omit to start now |
| `reservation_end_time` | string | No | Scheduled window end (RFC 3339) |
| `volume_id` | string | No | Existing shared volume ID to attach |
| `shared_volume` | object | No | Inline volume: `{volume_name, size_tib, region}` (project follows the cluster) |

## List Clusters

```python
response = client.clusters.list()
for c in response.clusters:
    print(f"{c.cluster_id}: {c.cluster_name} ({c.status}, {c.num_gpus} GPUs)")
```

```typescript
const response = await client.clusters.list();
for (const c of response.clusters) {
  console.log(`${c.cluster_id}: ${c.cluster_name} (${c.status})`);
}
```

```shell
curl -X GET \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  https://api.together.xyz/v1/compute/clusters
```

```shell
tg clusters list
```

## Get Cluster

```python
cluster = client.clusters.retrieve("cluster-id")
print(f"Status: {cluster.status}, GPUs: {cluster.num_gpus}")
```

```typescript
const cluster = await client.clusters.retrieve("cluster-id");
console.log(cluster);
```

```shell
curl -X GET \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  https://api.together.xyz/v1/compute/clusters/${CLUSTER_ID}
```

```shell
tg clusters retrieve <CLUSTER_ID>
```

## Update / Scale Cluster

```python
cluster = client.clusters.update(
    "cluster-id",
    num_gpus=16,
    cluster_type="KUBERNETES",
)
```

```typescript
const cluster = await client.clusters.update("cluster-id", {
  num_gpus: 16,
  cluster_type: "KUBERNETES",
});
```

```shell
curl -X PUT \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"num_gpus": 16, "cluster_type": "KUBERNETES"}' \
  https://api.together.xyz/v1/compute/clusters/${CLUSTER_ID}
```

```shell
tg clusters update <CLUSTER_ID> --num-gpus 16
tg clusters update <CLUSTER_ID> --num-gpus 16 --cluster-type KUBERNETES
```

### Update Request Body

| Field | Type | Description |
|-------|------|-------------|
| `num_gpus` | integer | New GPU count (must be a multiple of 8) |
| `cluster_type` | string | `KUBERNETES` or `SLURM` |

## Delete Cluster

```python
client.clusters.delete("cluster-id")
```

```typescript
await client.clusters.delete("cluster-id");
```

```shell
curl -X DELETE \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  https://api.together.xyz/v1/compute/clusters/${CLUSTER_ID}
```

```shell
tg clusters delete <CLUSTER_ID>
```

## List Regions

```python
regions = client.clusters.list_regions()
for r in regions.regions:
    print(f"{r.name}: {r.supported_instance_types}, drivers: {r.driver_versions}")
```

```typescript
const regions = await client.clusters.listRegions();
console.log(regions);
```

```shell
curl -X GET \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  https://api.together.xyz/v1/compute/regions
```

```shell
tg clusters list-regions
```

## Create Shared Volume

```python
volume = client.clusters.storage.create(
    volume_name="my-shared-volume",
    size_tib=2,
    region="us-central-8",
)
print(volume.volume_id)
```

```typescript
const volume = await client.clusters.storage.create({
  volume_name: "my-shared-volume",
  size_tib: 2,
  region: "us-central-8",
});
console.log(volume.volume_id);
```

```shell
curl -X POST \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"volume_name": "my-shared-volume", "size_tib": 2, "region": "us-central-8"}' \
  https://api.together.xyz/v1/compute/clusters/storage/volumes
```

```shell
tg clusters storage create \
  --volume-name my-shared-volume \
  --size-tib 2 \
  --region us-central-8
```

### Volume Create Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `volume_name` | string | Yes | Name of the volume |
| `size_tib` | integer | Yes | Size in tebibytes (TiB) |
| `region` | string | Yes | Region name |
| `project_id` | string | No | Project that owns the volume; default is the caller's default project. Ignored when the volume is created inline with a cluster. |

## List Shared Volumes

```python
volumes = client.clusters.storage.list()
for v in volumes.volumes:
    print(f"{v.volume_id}: {v.volume_name} ({v.size_tib} TiB, {v.status})")
```

```typescript
const volumes = await client.clusters.storage.list();
console.log(volumes);
```

```shell
curl -X GET \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  https://api.together.xyz/v1/compute/clusters/storage/volumes
```

```shell
tg clusters storage list
```

## Get Shared Volume

```python
volume = client.clusters.storage.retrieve("volume-id")
print(f"{volume.volume_name}: {volume.size_tib} TiB ({volume.status})")
```

```typescript
const volume = await client.clusters.storage.retrieve("volume-id");
console.log(volume);
```

```shell
curl -X GET \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  https://api.together.xyz/v1/compute/clusters/storage/volumes/${VOLUME_ID}
```

```shell
tg clusters storage retrieve <VOLUME_ID>
```

## Update (Resize) Shared Volume

```python
volume = client.clusters.storage.update(
    volume_id="volume-id",
    size_tib=5,
)
```

```typescript
const volume = await client.clusters.storage.update({
  volume_id: "volume-id",
  size_tib: 5,
});
```

```shell
curl -X PUT \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"volume_id": "volume-id", "size_tib": 5}' \
  https://api.together.xyz/v1/compute/clusters/storage/volumes
```

## Delete Shared Volume

Volume must not be attached to any cluster.

```python
client.clusters.storage.delete("volume-id")
```

```typescript
await client.clusters.storage.delete("volume-id");
```

```shell
curl -X DELETE \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  https://api.together.xyz/v1/compute/clusters/storage/volumes/${VOLUME_ID}
```

```shell
tg clusters storage delete <VOLUME_ID>
```

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

Available CUDA driver versions (check `list_regions()` in Python or `listRegions()` in TypeScript for per-region availability).

The `list_regions()` response returns driver versions as a list of objects:

```json
[
  {"cuda_version": "12.6", "nvidia_driver_version": "560"},
  {"cuda_version": "12.4", "nvidia_driver_version": "550"}
]
```

The combined `driver_version` string follows the pattern `CUDA_{major}_{minor}_{nvidia}`:

| `driver_version` | `cuda_version` | `nvidia_driver_version` |
|-------------------|----------------|--------------------------|
| `CUDA_12_4_550` | `12.4` | `550` |
| `CUDA_12_5_555` | `12.5` | `555` |
| `CUDA_12_6_560` | `12.6` | `560` |
| `CUDA_12_6_565` | `12.6` | `565` |
| `CUDA_12_8_570` | `12.8` | `570` |
| `CUDA_12_9_575` | `12.9` | `575` |

## Cluster Statuses

| Status | Description |
|--------|-------------|
| `Scheduled` | Cluster creation accepted, awaiting resource allocation |
| `WaitingForControlPlaneNodes` | Control plane provisioning |
| `WaitingForDataPlaneNodes` | Worker nodes provisioning |
| `WaitingForSubnet` | Network setup |
| `WaitingForSharedVolume` | Storage provisioning |
| `InstallingDrivers` | CUDA driver installation |
| `RunningAcceptanceTests` | GPU/network health validation |
| `Ready` | Cluster operational |
| `Degraded` | Some nodes unhealthy |
| `Paused` | Cluster paused |
| `OnDemandComputePaused` | On-demand compute paused (credit issue) |
| `Deleting` | Cluster being removed |

## Volume Statuses

| Status | Description |
|--------|-------------|
| `available` | Ready for attachment |
| `bound` | Attached to a cluster |
| `provisioning` | Being created |

## Cluster Response Object

```json
{
  "cluster_id": "abc-123-def-456",
  "cluster_name": "my-gpu-cluster",
  "cluster_type": "KUBERNETES",
  "region": "us-central-8",
  "gpu_type": "H100_SXM",
  "num_gpus": 8,
  "driver_version": "CUDA_12_6_560",
  "duration_hours": 720,
  "status": "Ready",
  "control_plane_nodes": [...],
  "gpu_worker_nodes": [...],
  "volumes": [...],
  "kube_config": "..."
}
```
