# Together AI: Dedicated Containers

Run your own Docker image on Together GPUs. Two deploy paths:

1. **Custom Sprocket worker** — implement `setup`/`predict`, build and push with Jig, submit
   async jobs to the managed queue.
2. **Prebuilt image** — set `image` (and usually `command`) under `[tool.jig.deploy]` to deploy
   an off-the-shelf image such as `vllm/vllm-openai` with no Sprocket code and no Docker build.
   Call the container's HTTP API directly (OpenAI client or curl), not `jig submit`.

Choose this only when the runtime is genuinely custom or you need a stock serving image on
reserved GPUs; a Together-hosted model belongs on `domains/dedicated-model-inference.md`.

## Workflow

### Custom worker (queue)

1. Confirm a custom runtime is required.
2. Implement the worker with Sprocket (`setup` once, `predict` per request).
3. Configure build (`[tool.jig.image]`), runtime, autoscaling, and mounts in `pyproject.toml`.
4. Build and deploy with `tg beta jig deploy`.
5. Submit jobs to the queue and poll each to completion; tear down the deployment when done.

### Prebuilt image (direct HTTP)

1. In `[tool.jig.deploy]`, set `image` to a pinned tag and `command` to the process to run
   (match `port` to the listen port; match `gpu_count` to tensor parallelism when applicable).
2. Optionally configure `[tool.jig.deploy.autoscaling]` (for vLLM, `CustomMetric` on a
   Prometheus metric such as `vllm:num_requests_running`).
3. Deploy with `tg beta jig deploy --detach` when model load can exceed the default ready wait,
   then watch `tg beta jig logs --follow` / `tg beta jig status` until Ready.
4. Call `https://api.together.ai/v1/deployment-request/<DEPLOYMENT_NAME>/...` with
   `TOGETHER_API_KEY` (for vLLM OpenAI routes, append `/v1`).
5. Tear down with `tg beta jig destroy` when done.

## Rules

- `pyproject.toml` is the source of truth for deployment behavior.
- Queue jobs are asynchronous; client code must poll and fetch results. Prebuilt HTTP servers
  are not queue workers — do not use `jig submit` against them.
- Pin image tags; `latest` can change on replica restart.
- Deployments bill while running; scale down or delete what you create.
- Parameterize deployment name, inputs, and resource sizing. `[project].name` is the deployment
  name and the last segment of the endpoint URL; it must be unique.
- While logged in with `tg login`, the CLI ignores `TOGETHER_API_KEY`; pass
  `--api-key "$TOGETHER_API_KEY"` on the first deploy (the platform stores a key on the
  deployment).

## Open next

| File | Lines | Contains | Open when |
|---|---|---|---|
| [scripts/dedicated-containers/sprocket_hello_world.py](scripts/dedicated-containers/sprocket_hello_world.py) | 85 | minimal Sprocket worker | writing a worker |
| [scripts/dedicated-containers/queue_client.py](scripts/dedicated-containers/queue_client.py) (.ts) | 109 | submit a job, poll, fetch the result | calling a deployed queue worker |
| [references/dedicated-containers/sprocket-sdk.md](references/dedicated-containers/sprocket-sdk.md) | 189 | Sprocket base class, `run`, file outputs, HTTP endpoints, env vars | a worker API detail |
| [references/dedicated-containers/jig-cli.md](references/dedicated-containers/jig-cli.md) | 546 | build, deploy (`--image`, `--detach`), queue, secrets, volumes; `pyproject.toml` including `image` and `[tool.jig.deploy.autoscaling]` | deploying or configuring |

## Docs

- [Dedicated Container Inference](https://docs.together.ai/docs/dedicated-container-inference)
- [Containers Quickstart](https://docs.together.ai/docs/containers-quickstart)
- [Deploy a prebuilt vLLM image](https://docs.together.ai/docs/dedicated-containers-vllm)
- [Deployments API](https://docs.together.ai/reference/deployments-create)
