---
name: Operate a self-hosted OpenSERP server
description: Bring up the MIT-licensed OpenSERP server, verify it is healthy, and read its cache,
  proxy-pool and circuit-breaker telemetry to diagnose why an engine has stopped returning results.
api: openapi/openserp-oss-openapi.yml
operations: [healthCheck, readinessCheck, listMegaEngines, getStats, getCacheStats, getProxyStats,
  getCircuitBreakerStats, getOpenAPISpec]
---

# Operate a self-hosted OpenSERP server

The self-hosted server is the free path and the sandbox path — it needs no API key, and the SDKs
point at it by changing one base URL. It is also the surface with the operational telemetry that
Cloud does not expose.

## Bring it up

```
docker run --rm -p 127.0.0.1:7000:7000 karust/openserp:latest
```

Or install the binary and run the server:

```
go install github.com/karust/openserp@latest
```

Advanced proxy pools, cache settings and other runtime parameters live in `config.yaml`. Port 7000
is the default and is settable with `-p`.

## Verify it

1. **`healthCheck`** (`GET /health`) — service health. Returns 503 when unhealthy.
2. **`readinessCheck`** (`GET /ready`) — readiness. Use this one as your container readiness probe;
   use `/health` as the liveness probe.
3. **`listMegaEngines`** (`GET /mega/engines`) — available engines and their runtime state. This is
   the check that tells you whether a specific engine is usable right now.
4. **`getOpenAPISpec`** (`GET /openapi.yaml`) — the server serves its own contract. `GET /docs`
   serves a Swagger UI against it. Diff this against
   `openapi/openserp-oss-openapi.yml` to confirm which version you are running.

## Diagnose a failing engine

When searches start returning `meta.engine_errors` entries, work down this list:

1. **`getCircuitBreakerStats`** (`GET /stats/cb`) — circuit-breaker state per engine. A
   `circuit_open` error class in a search response means the breaker has tripped for that engine;
   this endpoint tells you which and for how long. Back that engine off rather than retrying it.
2. **`getProxyStats`** (`GET /stats/proxy`) — proxy pool and per-engine proxy policy. Check here for
   `proxy_connect`, `proxy_auth`, `proxy_timeout` and `proxy_unavailable` error classes.
3. **`getCacheStats`** (`GET /stats/cache`) — cache statistics. Compare with the `X-Cache` response
   header (`HIT`, `MISS`, `BYPASS`) on individual searches.
4. **`getStats`** (`GET /stats`) — all three combined, if you want one call.

`captcha_detected` and `blocked` are not server faults — the engine refused the request. That is a
proxy or request-shape problem, not something the breaker will heal.

## Per-request proxy control

The server accepts request-scoped proxy headers, intended for a deployment sitting behind an
upstream balancer:

- `X-Use-Proxy` — override or disable the proxy for this request (`direct`, or a tag).
- `X-Proxy-URL` — a specific proxy URL supplied per request.
- `X-Proxy-Country`, `X-Proxy-Class` (`datacenter`, `residential`, `mobile`), `X-Proxy-Provider` —
  metadata describing the supplied proxy.
- `X-Proxy-Session-ID` — sticky session identifier; reuse the same value to keep a session.
- `X-Tenant` — namespaces sticky lane state in a multi-tenant deployment.

Responses echo what was actually used: `X-Proxy-Mode`, `X-Proxy-Tag`, `X-Proxy-Used` (a masked
`scheme://host:port`, never credentials), and `X-Browser-Profile-Id` for browser-mode runs.

## Rules

- **The self-hosted server has no authentication.** The published OpenAPI declares `security: []`
  and no security schemes. Bind it to `127.0.0.1` (as the documented Docker command does) or put
  your own gateway in front of it. Do not expose port 7000 to the internet.
- Correlate everything with `X-Request-ID` (UUID v7), which also appears as `meta.request_id` in
  the body and in the server logs.
- `X-Network-Bytes` reports inbound bytes consumed executing a search — the number to watch for
  egress cost. Cache hits report `0`.
- There is no rate limit on the self-hosted server. The 429 `rate_limited` class you see in the
  error schema applies to Cloud.
