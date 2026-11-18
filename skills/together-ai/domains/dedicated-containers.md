# Together AI: Dedicated Containers

Run your own Docker image as an inference worker on Together GPUs: Sprocket handles the request
lifecycle, Jig builds and deploys, clients submit async jobs to a queue, and an alpha routing
layer can forward HTTP, gRPC, and WebSocket traffic to ready replicas. Choose this only when
the runtime is genuinely custom; a standard model belongs on
`domains/dedicated-model-inference.md`.

## Workflow

1. Confirm a custom runtime is required.
2. Implement the worker with Sprocket (`setup` once, `predict` per request).
3. Configure image, runtime, autoscaling, and mounts in `pyproject.toml`.
4. Build and deploy with `jig`.
5. Submit jobs to the queue and poll each to completion; or call the alpha request-routing
   endpoint for sync HTTP/gRPC/WebSocket traffic. Tear down the deployment when done.

## Rules

- `pyproject.toml` is the source of truth for deployment behavior.
- Queue jobs are asynchronous; client code must poll and fetch results. Queue traffic does not
  use the request-routing layer.
- Request routing is alpha: endpoint host is privately shared; use
  `/v1/deployment-request/{name}` features (`together-affinity-key`, `together-replica`, gRPC,
  WebSockets) only on that URL.
- Only health-check-ready replicas receive routed requests; set
  `health_check_path` to `"/grpc.health.v1.Health"` for gRPC-only servers.
- Deployments bill while running; scale down or delete what you create.
- Parameterize deployment name, inputs, and resource sizing.

## Open next

| File | Lines | Contains | Open when |
|---|---|---|---|
| [scripts/dedicated-containers/sprocket_hello_world.py](scripts/dedicated-containers/sprocket_hello_world.py) | 85 | minimal Sprocket worker | writing a worker |
| [scripts/dedicated-containers/queue_client.py](scripts/dedicated-containers/queue_client.py) (.ts) | 109 | submit a job, poll, fetch the result | calling a deployed worker |
| [references/dedicated-containers/sprocket-sdk.md](references/dedicated-containers/sprocket-sdk.md) | 189 | Sprocket base class, `run`, file outputs, HTTP endpoints, env vars | a worker API detail |
| [references/dedicated-containers/jig-cli.md](references/dedicated-containers/jig-cli.md) | 529 | build, deploy, queue, secrets, volumes commands; `pyproject.toml` schema; full example | deploying or configuring |
| [references/dedicated-containers/request-routing.md](references/dedicated-containers/request-routing.md) | 137 | alpha routing URL, path strip, auth, affinity, pinning, gRPC/WebSockets, errors | sync HTTP/gRPC/WS to replicas |

## Docs

- [Dedicated Container Inference](https://docs.together.ai/docs/dedicated-container-inference)
- [Containers Quickstart](https://docs.together.ai/docs/containers-quickstart)
- [Request routing (alpha)](https://docs.together.ai/docs/deployments-request-routing)
- [Deployments API](https://docs.together.ai/reference/deployments-create)
