# Together AI: GPU Clusters

GPU clusters (H100, H200, B200, B300, and other types; availability varies by region) with
Kubernetes or Slurm, shared storage, and credentials, for distributed training, multi-node
inference, and HPC. Sizes are multiples of 8 GPUs. `ON_DEMAND` bills from creation until you
delete the cluster; `RESERVED` is charged for the full reserved term (3 to 180 days);
`SCHEDULED_CAPACITY` draws GPUs from a project capacity pool for a date window. Shared storage
bills separately and keeps billing after the cluster is deleted.

Prefer `tg clusters ...` and `client.clusters` (CLI/SDK 2.40+). CLI 3.0 removed `tg beta clusters`.
Older SDKs still expose `client.beta.clusters` as an alias. This skill's scripts keep the beta
client path where it still works so older SDK installs succeed.

Hand-offs: serving a model is `domains/dedicated-model-inference.md`; a custom inference container
is `domains/dedicated-containers.md`; short remote Python is `domains/sandboxes.md`; queueing or
gang scheduling on a running cluster is `domains/kueue.md` or `domains/volcano.md`.

## Workflow

1. Decide whether the workload really needs cluster-level control.
2. Choose on-demand, reserved, or scheduled-capacity billing based on run duration and pool access.
3. Choose Kubernetes vs Slurm based on orchestration requirements and team tooling.
4. Select region, GPU type, driver/CUDA (or `--nvidia-version-id`), and shared storage plan.
5. Provision first, then layer in access credentials, workload deployment, scaling, and health checks.

## Rules

- Prefer managed products unless the user explicitly needs raw infrastructure control.
- Treat storage lifecycle separately from cluster lifecycle; volumes can outlive clusters.
- When creating a cluster with new shared storage, prefer inline `shared_volume` over creating a volume separately and attaching via `volume_id`. Separately created volumes may land in a different datacenter partition than the cluster, causing a "does not exist in the datacenter" error even when the volume shows as available.
- A volume created alone belongs to your default project unless you pass `project_id`. A volume created with a cluster belongs to the cluster's project; `shared_volume.project_id` in the create request is ignored.
- GPU stock-outs (409 "Out of stock") are common. Always call `list_regions()` first and be prepared to try multiple regions.
- The regions response reports supported configurations, not prices or
  guaranteed stock. Never infer the cheapest GPU from response order or
  hardware generation; open the [GPU Cluster pricing table](https://www.together.ai/pricing#gpu-clusters)
  and compare its numeric on-demand rates.
- GPU clusters have an 8-GPU minimum. For an hourly estimate, multiply the displayed per-GPU-hour rate by at least 8.
- `ON_DEMAND` has no one-hour duration flag. For a one-hour plan, omit
  `--duration-days`, create the cluster only when authorized, then delete it
  after the intended runtime. If the user requests a command without
  provisioning, print it but do not run it.
- `SCHEDULED_CAPACITY` requires a project capacity pool (`capacity_pool_id`). Use
  `num_capacity_pool_gpus` for the pool share; for a hybrid start-now cluster also set
  `num_reserved_gpus`. Omit the start time to start immediately.
- The API requires `cuda_version` and `nvidia_driver_version` (and optionally `os`; default
  `ubuntu-22.04`). Current SDKs accept them as first-class fields; older Python clients may
  still need `extra_body`. Or pass `--nvidia-version-id` / the catalog `id` from `list-regions`.
- Interactive `tg clusters delete` and `tg clusters storage delete` confirm before deleting.
  Pass `--force`, or the global `--non-interactive` / `--json` flags, in scripts and CI.
- Credentials retrieval is part of provisioning. Do not stop at cluster creation if the user needs to run workloads immediately. Only project admins can download the admin kubeconfig.
- Node repair: auto repair uses reboot, quick reprovision (`VM_ONLY`), and migrate
  (`HOST_AWARE`). Manual Cycle (`HOST_POWER_CYCLE`) is account/hardware gated and returns
  `403` when unavailable. Drive remediations with `tg clusters remediations ...`.
- Slurm startup scripts (worker/login init, worker/controller prolog and epilog, extra `slurm.conf`) are **Slinky v1.0 only**. A non-zero exit from a worker prolog or epilog drains the node, and calling Slurm commands (`squeue`, `scontrol`, `sacctmgr`) inside any prolog/epilog can deadlock the scheduler.

## Open next

For read-only questions (regions, GPU types, prices, a command to run later), the 82-line
`pricing-and-discovery.md` is all you need. Open the others only for the detail in their row, and
read just that section; long ones start with `## Contents`.

| File | Lines | Contains | Open when |
|---|---|---|---|
| [references/gpu-clusters/pricing-and-discovery.md](references/gpu-clusters/pricing-and-discovery.md) | 82 | authoritative pricing and stock surfaces, read-only inventory commands, picking the cheapest live option, one-hour on-demand plan | pricing, availability, or a plan without provisioning |
| [scripts/gpu-clusters/manage_cluster.py](scripts/gpu-clusters/manage_cluster.py) (.ts) | 237 | create, poll, credentials, scale, delete | provisioning or changing a cluster |
| [scripts/gpu-clusters/manage_storage.py](scripts/gpu-clusters/manage_storage.py) | 138 | create, list, resize, delete shared volumes | managing storage on its own |
| [references/gpu-clusters/cli.md](references/gpu-clusters/cli.md) | 201 | `tg clusters` create/update/delete/storage/SSH/remediations, global flags, instance types | a CLI flag, delete confirmation, or repair mode |
| [references/gpu-clusters/api-reference.md](references/gpu-clusters/api-reference.md) | 490 | cluster and storage endpoints, scheduled-capacity fields, regions, instance types | an API field the script does not set |
| [references/gpu-clusters/cluster-management.md](references/gpu-clusters/cluster-management.md) | 597 | architecture, SSH, Slurm, storage, health checks, Cycle repair, remediations CLI, billing, troubleshooting | operating or debugging a running cluster |

## Docs

- [GPU Clusters Overview](https://docs.together.ai/docs/gpu-clusters-overview)
- [GPU Clusters Quickstart](https://docs.together.ai/docs/gpu-clusters-quickstart)
- [Manage GPU clusters](https://docs.together.ai/docs/gpu-clusters-management)
- [Run workloads on Kubernetes](https://docs.together.ai/docs/kubernetes-workloads)
- [Connect over SSH](https://docs.together.ai/docs/gpu-clusters-ssh)
- [Automate cluster management](https://docs.together.ai/docs/gpu-clusters-automation)
- [Node repair](https://docs.together.ai/docs/node-repair)
- [Clusters CLI](https://docs.together.ai/reference/cli/clusters)
- [Clusters API](https://docs.together.ai/reference/clusters-create)
- [GPU Cluster Pricing](https://www.together.ai/pricing#gpu-clusters)
- [Slurm Startup Scripts](https://docs.together.ai/docs/slurm-startup-scripts)
- [Instant GPU Clusters](https://www.together.ai/instant-gpu-clusters)
