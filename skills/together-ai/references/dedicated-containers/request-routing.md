# Request routing (alpha)

## Contents

- [Overview](#overview)
- [Endpoint URL](#endpoint-url)
- [Path forwarding](#path-forwarding)
- [Authentication](#authentication)
- [Replica selection](#replica-selection)
- [Session affinity](#session-affinity)
- [Replica pinning](#replica-pinning)
- [Shutdown](#shutdown)
- [gRPC](#grpc)
- [WebSockets](#websockets)
- [Errors](#errors)

## Overview

The dedicated container inference **routing layer** (alpha; can change without notice)
authenticates each request and forwards it to one ready replica. It carries HTTP, gRPC, and
WebSocket traffic. Queue API jobs do **not** pass through it.

The alpha endpoint URL is not published yet; contact sales to get it.

## Endpoint URL

Every deployment has an alpha URL ending in `/v1/deployment-request/{DEPLOYMENT_NAME}`. It has
the same path as `jig endpoint` prints but a different host. Only the alpha URL supports the
features below.

```bash
export ENDPOINT_URL="https://<alpha-host>/v1/deployment-request/<DEPLOYMENT_NAME>"
```

## Path forwarding

The routing layer strips `/v1/deployment-request/{DEPLOYMENT_NAME}` and forwards the rest of the
path (percent-encoding preserved). Method, body, and query string arrive unchanged.

| Request path | Path the container receives |
|---|---|
| `.../my-model/generate` | `/generate` |
| `.../my-model/v1/chat/completions?stream=true` | `/v1/chat/completions?stream=true` |
| `.../my-model` | `/` |

HTTP request bodies are capped at 32 MiB. gRPC has no size limit. Responses stream as the
container writes them (SSE works without extra config).

## Authentication

Send a Together API key from the deployment's project:

```bash
curl "$ENDPOINT_URL/health" \
  -H "Authorization: Bearer $TOGETHER_API_KEY"
```

The routing layer verifies the key before forwarding. The container does not need to validate it.

## Replica selection

1. Pick a cluster at random among those running the deployment, weighted by ready replica count.
2. Pick one ready replica in that cluster in round-robin order.

Neither stage accounts for load. Busy replicas get new requests at the same rate as idle ones.
Limit concurrency in the server, or send long-running work through the queue instead.

Only replicas that pass the health check receive requests. The check uses
`health_check_path` (default `/health`): return `200` when ready, `503` when not.

## Session affinity

Send the same `together-affinity-key` on related requests to prefer one replica (for example
cached prompt prefixes). The key must be printable ASCII; hash non-ASCII values first.

Affinity is best effort. When the set of ready replicas changes (deploy, scale, crash), some
keys move: keys on a replica that stops being ready move elsewhere; a newly ready replica takes
a share of keys; cluster weighting can move a key across clusters.

## Replica pinning

Set `together-replica` to a replica ID (keys of `replica_events` from the get-deployment API).
Pinned requests are never redirected. If the replica is not ready or gone, the response is
`410`. Replica IDs change on replace; refetch after deploy/scale-down and on any `410`.

When both headers are set, `together-replica` wins over `together-affinity-key`.

```bash
curl "$ENDPOINT_URL/health" \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  -H "together-replica: <REPLICA_ID>"
```

## Shutdown

On deploy or scale-down, a shutting-down replica stops receiving new requests and gets
`SIGTERM`, then has `termination_grace_period_seconds` to finish in-flight work. Handle
`SIGTERM` and set the grace period longer than the longest request.

The routing layer can also drop long-lived connections (WebSockets, gRPC streams) during its
own maintenance; clients should reconnect.

## gRPC

Together terminates TLS in the replica's cluster and forwards to the container over HTTP/2
(h2c) on `port`. Treat a request as gRPC when `Content-Type` is `application/grpc` (optional
`+proto` or `;` parameters).

Connect to the endpoint host on port 443 with a secure channel. Every gRPC method path must
start with `/v1/deployment-request/{DEPLOYMENT_NAME}`; rewrite with a client interceptor. The
routing layer strips the prefix before forwarding.

For a gRPC-only server, implement the standard gRPC health service and set
`health_check_path = "/grpc.health.v1.Health"` (default readiness is HTTP/1.1). Routing-layer
gRPC rejections return a gRPC status on HTTP `200`.

## WebSockets

Supports HTTP/1.1 `Upgrade` only (not WebSockets over HTTP/2). Path prefix stripping matches
HTTP. The API key must be in the `Authorization` header on the upgrade request; browser
WebSocket APIs cannot set it, so use a server-side client (`wss://` URL).

## Errors

Routing-layer rejections have an empty body; container error responses pass through unchanged.
Give container errors a body to tell them apart.

| HTTP | gRPC | Cause | Action |
|---|---|---|---|
| `400` | `2` UNKNOWN | Affinity key outside printable ASCII | Use printable ASCII or hash first |
| `401` | `16` UNAUTHENTICATED | Missing/invalid bearer key | Send a valid Together API key |
| `403` | `7` PERMISSION_DENIED | Key from a different project | Use a key from the deployment's project |
| `404` | `12` UNIMPLEMENTED | Bad path prefix or deployment not running | Check name and gRPC method prefix |
| `410` | `5` NOT_FOUND | Pinned replica not ready/gone, or non-ASCII replica id | Refetch replica IDs or drop the header |
| `413` | — | HTTP body over 32 MiB | Send a smaller body |
| `502` | `14` UNAVAILABLE | No response from replica | Check `port`/protocol and `jig logs`; retry |
| `503` | `14` UNAVAILABLE | No ready replica, or key verification failed | Check `jig status` and health; retry |
