# Changelog

All notable changes to this repository are recorded here. No version
tags are used — this repo tracks Phase 3 (Atlas Observability)
progress by date instead, consistent with the roadmap's phase-based
structure rather than a released package's semver history.

## 2026-09-09

- `fix(dashboard)`: guarded the Error Rate panel's PromQL against
  divide-by-zero on empty traffic windows.
- `fix(dashboard)`: replaced the stale "Saturation — not yet
  instrumented" placeholder panel with real event-loop utilization
  and V8 heap-used panels — the metrics had been flowing all along
  (`auto-instrumentations-node`'s bundled `instrumentation-runtime-node`),
  the dashboard just hadn't been updated to reflect it.

## 2026-09-07

- `fix(otel-collector)`: dropped `instrumentation-http`'s
  `http.client.request.duration` at the collector level to resolve a
  Prometheus HELP-string collision with `instrumentation-undici` —
  see [ADR-0004](docs/adr/ADR-0004-prometheus-http-client-duration-collision.md)
  and [the evidence doc](docs/evidence/prometheus-client-duration-collision-fix.md).

## 2026-09-04

- `docs(runbook)`: wrote the incident runbook, grounded in real alert
  rules and the threat model rather than generic scenarios.
- `docs(readme)`: added Incident Runbook and Threat Model summary
  sections.
- `docs(threat-model)`: full STRIDE analysis grounded in this repo's
  actual config and session evidence.
- `feat(alerting)`: wired Alertmanager to Slack with severity-based
  routing (critical → 1h repeat, warning → 4h repeat).
- `docs(readme)`: reflected the Golden Signals dashboard, alerting,
  and ADR-0003 in the README.

## 2026-09-02

- `feat(dashboards)`: added the Golden Signals dashboard for Tawira
  (request rate, error rate, p50/p95/p99 latency, status codes,
  saturation, Collector pipeline health).

## 2026-09-01

- `docs`: added [ADR-0003](docs/adr/ADR-0003-otel-batch-processor-constructor-signature.md)
  for the OTel `Batch*Processor` constructor-signature breaking
  change (positional args → options object in OTel v2.x).
- `fix(observability)`: committed a missing `prometheus/alert-rules.yml`.
- `fix(otel-collector)`: exposed collector self-telemetry on
  `0.0.0.0:8888` (was bound to localhost only inside the container,
  making it unreachable from the host for verification).

## 2026-08-31

- `fix(security)`: moved the Grafana admin password out of
  `docker-compose.yml`.

## 2026-08-28

- `docs`: fixed a truncated README, added
  [ADR-0001](docs/adr/ADR-0001-use-tawira-instead-of-sample-app.md)
  and [ADR-0002](docs/adr/ADR-0002-loki-tempo-ring-readiness-race.md),
  captured initial stack-health evidence.
- `docs`: added `CONTRIBUTING.md`.
- `feat`: initial LGTM stack scaffold — Prometheus, Loki, Tempo,
  Grafana, Alertmanager, OTel Collector via Docker Compose.