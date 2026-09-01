# ADR-0003: Pass Exporters via Options Object to Batch*Processor Constructors

## Status

Accepted

## Date

2026-09-01

---

## Context

Tawira's OpenTelemetry instrumentation (`instrumentation.mjs`) wires trace, log, and metric export to atlas-observability's Collector. During verification, the Collector's own self-telemetry (`otelcol_receiver_accepted_*` counters) showed metric points arriving correctly, but zero spans and zero log records — despite `instrumentation-http` correctly patching Node's `http` module and correctly creating spans for every request (confirmed via a temporary `ConsoleSpanExporter` tap, independent of the OTLP export path).

This ruled out several plausible causes before the real one was found:

- **Vercel-dev process isolation.** Tawira's Nitro config defaults to the `vercel` preset even in `vite dev`, and requests showed a `server: Vercel` response header. This raised a real possibility that requests were handled by a separate, uninstrumented process. Confirmed false: the console-exporter tap showed a correctly propagated distributed trace across both the Vite dev process and a spawned `env-runner` worker process, with correct parent/child span linkage and W3C trace-context propagation across the process boundary.
- **Collector unreachable.** Ruled out early — metrics were exporting successfully over the same OTLP HTTP endpoint, so the network path and Collector receiver were confirmed working.

With `OTEL_DEBUG=1` diagnostics enabled, the actual failure surfaced on the first batch flush:

TypeError: Cannot read properties of undefined (reading 'export')
    at doExport (BatchSpanProcessorBase.js:142:55)

and identically for `BatchLogRecordProcessorBase.js`. Metrics were unaffected because `PeriodicExportingMetricReader` does not go through this code path at all.

Inspecting the published `.d.ts` and compiled source directly for the pinned versions (`@opentelemetry/sdk-trace@2.11.0`, `@opentelemetry/sdk-logs@0.222.0`) confirmed the cause: both `BatchSpanProcessor` and `BatchLogRecordProcessor` now take a single **options object** with an `exporter` property, not the exporter as a positional argument — a breaking change from the older v1.x API most existing examples and training-data knowledge still reflect. `instrumentation.mjs` was passing the exporter positionally, so `options.exporter` was `undefined` inside `BatchSpanProcessorBase`'s constructor. The bug did not surface until the first scheduled flush, because construction itself doesn't touch `_exporter`.

---

## Decision

Pass the exporter as `{ exporter }` to both `BatchSpanProcessor` and `BatchLogRecordProcessor`, matching the v2.x-era constructor signature confirmed against the actual published packages at the versions Tawira has pinned.

Separately, and unrelated in cause but discovered in the same debugging session: the Collector's own self-telemetry metrics endpoint defaults to binding `localhost:8888` inside the container (pre-`0.123.0` default, confirmed against OTel's own docs for the pinned `0.106.0` image), and `docker-compose.yml` did not publish that port to the host. `otelcol_receiver_accepted_*` — the exact counters this investigation depended on — were unreachable from outside the container until both the bind address and the port mapping were fixed. This is now `service.telemetry.metrics.address: 0.0.0.0:8888` in `otel-collector/config.yaml`, published as `8888:8888` in `docker-compose.yml`.

---

## Rationale

- Root-caused against the actual installed package source and `.d.ts`, not assumed from memory or older examples — the v1.x → v2.x constructor signature change is exactly the kind of drift that's easy to carry forward incorrectly from training data or older blog posts.
- Verified via empirical evidence at every layer, not just "it compiles": Collector self-telemetry counters (`otelcol_receiver_accepted_spans`, `_log_records`, `_metric_points`) confirmed non-zero, and Tempo's stored trace confirmed the full 4-span structure (root SSR span → outgoing client span → `tcp.connect` → nested worker-process span) rendering correctly in the Explore UI.
- Verified independently against **two** code paths — `vite dev` and the built Docker image (`node-server` Nitro preset) — since dev-mode success does not predict the production bundle's behavior, a risk this repo had already flagged before this fix.

---

## Alternatives Considered

### Downgrade to an older `@opentelemetry/sdk-trace`/`sdk-logs` major version with the positional-argument API

Advantages:

- No code change needed in `instrumentation.mjs`.

Disadvantages:

- Fights the ecosystem rather than adapting to it; the pinned versions were chosen deliberately (see the dependency table in the handoff notes) and downgrading reopens compatibility questions across the rest of the OTel dependency graph (`resourceFromAttributes`, `spanProcessors`/`metricReaders`/`logRecordProcessors` plural fields on `NodeSDK`, etc., all v2.x-era APIs).
- Delays inevitably hitting this same signature change later.

---

## Consequences

### Positive

- Traces and logs now export correctly, closing out Phase 3's instrumentation verification end-to-end.
- The debugging trail itself — ruling out a plausible-but-wrong theory (process isolation) with real evidence before finding the actual cause — is a stronger artifact than a first-try success would have been.
- Collector self-telemetry is now genuinely observable from the host, which will matter again for any future Collector-side debugging, not just this one.

### Negative

- None identified. The fix is a pure bug fix with no behavioral tradeoff.

---

## Review

Revisit if the pinned `@opentelemetry/sdk-trace` / `@opentelemetry/sdk-logs` major versions change again — v2.x constructor signatures should not be assumed stable across future majors either.