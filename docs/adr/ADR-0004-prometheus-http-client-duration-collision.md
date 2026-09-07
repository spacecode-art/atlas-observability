# ADR-0004: Drop instrumentation-http's http.client.request.duration to Resolve Prometheus Exporter Collision

## Status

Accepted

## Date

2026-09-04

---

## Context

During local Supabase verification (Tawira, Phase 3 closeout), `http_server_request_duration_seconds` — the metric Golden Signals depends on — returned empty from Prometheus even after confirming, via `OTEL_DEBUG=1`, that correctly-formed HTTP server spans and metrics were being generated and handed to the OTLP exporter for every request.

The Collector's own logs showed the real cause, recurring on every scrape:

    error gathering metrics: collected metric http_client_request_duration_seconds ...
    has help "Measures the duration of outbound HTTP requests."
    but should have "Duration of HTTP client requests."

Root-caused against the actual installed package source, not assumed:

- `@opentelemetry/instrumentation-http@0.222.0` (`build/src/http.js:43`) registers `http.client.request.duration` with description `"Duration of HTTP client requests."`
- `@opentelemetry/instrumentation-undici@0.32.0` (`build/src/undici.js:58`) registers a metric under the **same name** with description `"Measures the duration of outbound HTTP requests."`

Both are bundled and active via `auto-instrumentations-node@0.80.0`. `instrumentation-undici` fires because Tawira's Supabase client calls go through Node's global `fetch()` (undici-backed) — see `auth-middleware.ts`'s `createSupabaseFetch`. Prometheus's `client_golang` enforces exactly one HELP string per metric name across a scrape; the mismatch caused `Gather()` to fail for that metric family on every scrape.

Confirmed this was *not* the cause of the server-metric emptiness (that was a separate issue — see below): after the collector-side fix, `error gathering metrics` disappeared entirely from the logs, while `http_server_request_duration_seconds` had already been returning correct data even *before* this fix was applied, in the same scrape cycles that were failing on the client-duration family. `client_golang`'s `Gather()` evidently returns partial results per metric family rather than failing the whole response — worth correcting the initial hypothesis in this session that treated the two as one bug.

---

## Decision

Added a `filter` processor to `otel-collector/config.yaml`, matched at the `metrics.metric` OTTL context on both `name` and `description`, dropping only `instrumentation-http`'s copy of `http.client.request.duration` and leaving `instrumentation-undici`'s intact:

```yaml
filter/dedupe-client-duration:
  error_mode: ignore
  metrics:
    metric:
      - 'name == "http.client.request.duration" and description == "Duration of HTTP client requests."'
```

Wired into the metrics pipeline after `attributes/scrub`, before the `prometheus` exporter. Verified via `docker compose exec otel-collector cat /etc/otel-collector/config.yaml` against the actual mounted file (not just the local copy — a bind-mount timing gap during verification initially made this look unapplied when it wasn't), and via live query showing a single, correctly-attributed `http_client_request_duration_seconds` series against `xecomdfrdxbjccnldduv.supabase.co` with no further `error gathering` log lines.

---

## Rationale

- Root-caused against actual package source (`build/src/*.js`), not assumed from either package's README or training data.
- Fix targets the collector layer, not app code — doesn't touch Tawira's `instrumentation.mjs`, and doesn't disable either instrumentation wholesale (a blunter fix would have silenced `http.server.request.duration` too, since it shares `instrumentation-http`'s scope name).
- Kept `instrumentation-undici`'s version specifically because it's the one actually observing real traffic — Supabase's SDK never touches the core `http`/`https` modules that `instrumentation-http` patches, since Node's `fetch()` is undici-backed and bypasses that code path entirely.

---

## Alternatives Considered

### Disable `@opentelemetry/instrumentation-undici` entirely in `instrumentation.mjs`

Advantages:
- Simpler, single-line app-side change.

Disadvantages:
- Loses all outbound-call visibility for Supabase (and any other fetch-based) traffic, since `instrumentation-http` cannot see undici-backed requests at all. Solves the symptom by discarding the exact data this phase is supposed to demonstrate.

### Disable `@opentelemetry/instrumentation-http` entirely

Advantages:
- Also resolves the collision.

Disadvantages:
- `instrumentation-http` is the sole source of `http.server.request.duration` — the actual metric Golden Signals depends on. This would have reopened the original blocker.

### Pin a matching version of one package to align description text

Advantages:
- Fixes the root string mismatch directly rather than filtering around it.

Disadvantages:
- No version pair was confirmed to agree on this string at the time of investigation; would require ongoing vigilance against re-drift on every future bump of either package. The collector-side filter is more durable against this class of upstream inconsistency recurring.