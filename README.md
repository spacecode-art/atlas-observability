# Atlas Observability

> Full-stack observability for the Atlas platform — metrics, logs, and
> traces from a real application, self-hosted at zero cost.

## Problem Statement

Managed observability (Grafana Cloud, Datadog, hosted Prometheus)
hides the operational complexity it's solving — you get dashboards
without ever having to run the stack that produces them. Atlas
Observability builds the full LGTM stack (Prometheus, Grafana, Loki,
Tempo) self-hosted, instruments a real application with
OpenTelemetry, and proves the result with real telemetry, not
synthetic load — all provable without a hosted observability bill.

## A note on anonymization

The telemetry in this repo comes from Tawira, a production SaaS
application I run — not a sample app. Service names, route names, and
other identifying details are genericized (`atlas-demo-*`) because
Tawira is private; the metrics, traces, and error patterns themselves
are unmodified and real. Tawira runs locally against a local Supabase
instance for this repo's purposes — not against production data — so
instrumentation can be developed and tested without touching a live
system with real users. See
[ADR-0001](docs/adr/ADR-0001-use-tawira-instead-of-sample-app.md) for
why a real app was chosen over a throwaway one, consistent with the
same decision in
[`atlas-security`](https://github.com/spacecode-art/atlas-security).

## Overview

Atlas Observability is Phase 3 of the Atlas platform. It stands up a
self-hosted LGTM stack via Docker Compose, instruments a real
application with OpenTelemetry, and builds Golden Signals dashboards
and alerting rules against real (anonymized) traffic — proving the
stack works without a managed service doing the hard parts.

---

## Objectives

- Full LGTM stack (Prometheus, Grafana, Loki, Tempo) via Docker Compose
- OpenTelemetry instrumentation on Tawira (anonymized), covering
  metrics, structured logs, and distributed traces
- Golden Signals dashboards (latency, traffic, errors, saturation)
  built against real (anonymized) traffic
- Alertmanager rules, tested against synthetic/replayed data
- Document every non-trivial decision as an ADR, same discipline as
  `atlas-foundation` and `atlas-security`

---

## Architecture Diagram

```mermaid
graph TB
    subgraph "Tawira (local instance, anonymized)"
        APP[atlas-demo-api]
        DB[(local Supabase)]
        OTEL_SDK[OTel SDK<br/>metrics + logs + traces]
        APP --> OTEL_SDK
        APP --> DB
    end

    subgraph "Collection"
        COLLECTOR[OTel Collector<br/>attribute scrub + batch]
        OTEL_SDK -->|OTLP gRPC/HTTP| COLLECTOR
    end

    subgraph "LGTM Stack (Docker Compose, self-hosted)"
        PROM[Prometheus]
        LOKI[Loki]
        TEMPO[Tempo]
        GRAFANA[Grafana]

        COLLECTOR --> PROM
        COLLECTOR --> LOKI
        COLLECTOR --> TEMPO
        PROM --> GRAFANA
        LOKI --> GRAFANA
        TEMPO --> GRAFANA
    end

    subgraph "Alerting"
        ALERTMGR[Alertmanager]
        PROM --> ALERTMGR
    end
```

---

## Repository Structure

```text
atlas-observability/
├── .github/
│   └── workflows/            # CI (repository validation)
├── docs/
│   ├── adr/                  # Architecture Decision Records
│   ├── diagrams/              # Architecture diagrams
│   ├── evidence/               # Committed proof-of-work
│   ├── threat-model.md         # STRIDE threat model
│   └── incident-runbook.md     # Operational runbook
├── otel-collector/
│   └── config.yaml            # OTLP receivers, attribute scrub, exporters
├── prometheus/
│   └── prometheus.yml
├── loki/
│   └── loki-config.yaml
├── tempo/
│   └── tempo-config.yaml
├── alertmanager/
│   └── alertmanager.yml
├── dashboards/
│   └── grafana/
│       ├── provisioning/
│       │   ├── datasources/datasources.yml
│       │   └── dashboards/dashboard.yml
│       └── dashboards/         # golden-signals.json — live, verified against real Tawira traffic
├── .editorconfig
├── .gitignore
├── CHANGELOG.md
├── CONTRIBUTING.md
├── LICENSE
├── docker-compose.yml
└── README.md
```

---

## Technology Choices

| Choice | Why this, not the alternative |
|---|---|
| **Self-hosted LGTM stack** over Grafana Cloud/Datadog | Zero cost, and running the stack yourself is a stronger skill signal than consuming a managed dashboard. |
| **OpenTelemetry** over vendor-specific SDKs | Vendor-neutral instrumentation — the same OTel SDK config works whether the backend is this self-hosted stack or a managed one later. |
| **Tawira (real app)** over a throwaway sample app | Real, evolving traffic produces real, evolving telemetry — same reasoning as `atlas-security` ADR-0001. |
| **Local Tawira + local Supabase** over pointing at production | Zero risk to a real system with real users; instrumentation proof doesn't require live user traffic, only real code paths under real (self-generated) load. |
| **Attribute-scrub processor in the Collector** over relying on the SDK alone | Defense in depth — anonymization is enforced at the source (SDK config) *and* at the collector, so a misconfigured service name doesn't leak identifying data by itself. |

---

## Deployment Guide

**Prerequisites:** Docker, Docker Compose, Supabase CLI (for local Tawira).

```bash
# 1. Start the LGTM stack
docker compose up -d

# 2. Verify each service is healthy
docker compose ps

# 3. Grafana: http://localhost:3000 (anonymous viewer access enabled)
# 4. Prometheus: http://localhost:9090
# 5. Alertmanager: http://localhost:9093
```

Tawira instrumentation: wired in and verified end-to-end, both in
`vite dev` and the built Docker/production Nitro path — see Current
Status below.

---

## Current Status

**Built:**
- `docker-compose.yml` — full LGTM stack (Prometheus, Loki, Tempo,
  Grafana, Alertmanager) plus an OTel Collector with an
  attribute-scrubbing processor as a second anonymization layer
- Grafana provisioned with all three datasources (Prometheus, Loki,
  Tempo) pre-wired, including trace-to-logs and trace-to-metrics
  correlation, with explicit `uid`s pinned so that correlation
  can't silently break across Grafana versions
- Full stack verified running end-to-end: all six services confirmed
  healthy via readiness-endpoint checks, not just `docker compose ps`
  (see [`docs/evidence/stack-health-check.md`](docs/evidence/stack-health-check.md)).
  A startup readiness race in Loki/Tempo was investigated and found
  to be expected behavior, not a bug (ADR-0002)
- Tawira instrumented with the OTel SDK (traces, logs, metrics),
  including a real breaking-change bug found, fixed, and verified
  against a live Tempo trace (ADR-0003)
- Golden Signals dashboard, built against real (anonymized) Tawira
  traffic — request rate, error rate, p50/p95/p99 latency, status
  code breakdown, event-loop/heap saturation, and Collector pipeline
  health. Every metric name verified against live Prometheus data,
  not just published `.d.ts` source (see
  [`docs/evidence/golden-signals-verification.md`](docs/evidence/golden-signals-verification.md))
- Alertmanager: severity-routed to Slack (critical/warning), rules
  covering error rate, latency, event-loop saturation, and target
  liveness. Proven end-to-end with synthetic alerts confirmed
  delivered in Slack, not just config-valid (see
  [`docs/evidence/alertmanager-slack-verification.md`](docs/evidence/alertmanager-slack-verification.md))

**Not yet built, tracked honestly:**
- Local Supabase instance (via `supabase start`) not yet stood up
- Demo video

---

## Cost Model

**$0 spent.** The entire stack runs via Docker Compose on a local
machine — Prometheus, Loki, Tempo, Grafana, Alertmanager, and the OTel
Collector are all free/open-source, self-hosted. Tawira runs locally
against a local Supabase instance, not a hosted/paid Supabase project.

---

## Design Decisions (ADRs)

| ADR | Decision |
|---|---|
| 0001 | Use Tawira (private production SaaS) instead of a throwaway sample app |
| 0002 | Loki/Tempo ring-readiness startup race — accepted, not a bug |
| 0003 | Pass exporters via options object to `Batch*Processor` constructors — OTel v2.x breaking change from the older positional-argument API |

---

## Testing Strategy

Nothing in this repo is marked "done" on the strength of a config
file parsing cleanly. Every component was proven at the next level up:

- **Config validity**: `promtool check rules` / `amtool check-config`
  run against the exact pinned image versions (both require
  `--entrypoint` overrides — the images set the binary itself as
  `ENTRYPOINT`, so the check subcommand as a bare arg just errors).
- **Metric-shape verification**: every metric name used in the
  dashboard and alert rules was checked against the actual published
  npm tarball source for the pinned OTel package versions, then
  confirmed a second time against live Prometheus query output —
  matching this repo's founding lesson (ADR-0003) that library API
  shape should be verified, not assumed from memory or older docs.
- **End-to-end delivery, not just "config accepted"**: Alertmanager
  routing was proven by firing synthetic alerts through its real API
  and confirming both the correct receiver match *and* actual message
  delivery in Slack — a receiver name in an API response doesn't
  prove a webhook POST succeeded.
- **Both runtime paths**: Tawira's OTel instrumentation was verified
  in `vite dev` and in the built Docker/`node-server` Nitro path
  separately, since a dev-mode success doesn't predict the built
  bundle (this was an explicit risk called out before either was
  tested, and it caught nothing here — but the check itself is the
  point).

---

## Monitoring

Golden Signals dashboard (Grafana, auto-provisioned) covers:

- **Rate** — request rate by HTTP method
- **Errors** — 5xx ratio over a 5m window
- **Duration** — p50/p95/p99 latency via `histogram_quantile`
- **Saturation** — Node.js event-loop utilization and delay p99,
  V8 heap used (via `@opentelemetry/instrumentation-runtime-node`,
  bundled automatically by `auto-instrumentations-node`, no explicit
  wiring required)

Alerting (Alertmanager, routed to Slack by severity):

| Alert | Condition | Severity |
|---|---|---|
| `HighErrorRate` | 5xx ratio > 5% for 5m | critical |
| `HighLatencyP99` | p99 latency > 2s for 5m | warning |
| `EventLoopSaturation` | event-loop utilization > 0.9 for 5m | warning |
| `TargetDown` | any scrape target unreachable for 2m | critical |

Distributed tracing (Tempo, via Grafana Explore): confirmed rendering
a full nested trace across the Vite dev process and the spawned
`env-runner` worker process, with correct W3C trace-context
propagation across the process boundary — see ADR-0003 for the
investigation this came out of.

**Known gap**: the worker process's `service.name` resource attribute
reads `atlas-demo-apib` — a stray trailing `b` — instead of
`atlas-demo-api`. Confirmed present in both raw trace data and the
`exported_job` label on metrics (wider than originally scoped).
Logged, not yet fixed; next step is `grep -rn OTEL_SERVICE_NAME
node_modules/env-runner/dist/runners/vercel/worker.mjs`.

---

## Threat Model

Full STRIDE analysis: [`docs/threat-model.md`](docs/threat-model.md).
Grounded in this repo's actual config and session evidence, not
generic boilerplate — two findings were verified against upstream
source rather than assumed, and one (the unauthenticated Alertmanager
write API) was demonstrated directly via the synthetic-alert curls
used for the Slack routing proof.

**Every service in this stack is unauthenticated at the network
layer** — a deliberate, stated tradeoff for local-only deployment,
not an oversight.

| Finding | Category | Status |
|---|---|---|
| OTLP receivers accept unauthenticated ingest from any reachable client | Spoofing | Accepted risk, local-only |
| Alertmanager's write API has no auth — demonstrated this session via curl | Tampering | Accepted risk, local-only |
| Slack webhook URL pasted in plaintext during setup (real incident) | Information Disclosure | Fixed — routed through `api_url_file`, webhook rotated |
| `--web.enable-lifecycle` exposes unauthenticated `POST /-/quit`, verified against Prometheus v2.55.1 source | Denial of Service | Accepted risk, local-only |
| Grafana anonymous Viewer can run arbitrary PromQL/LogQL via Explore | Information Disclosure | Partially mitigated — Collector's `attributes/scrub` processor means no identifying data reaches Prometheus/Loki to query in the first place |

**What would actually need to change before any public exposure**:
auth extension on the OTLP receiver, a reverse proxy in front of
every service (Alertmanager, Prometheus, Loki, Tempo, Collector
self-telemetry), dropping `--web.enable-lifecycle`, and disabling
Grafana anonymous access. None of this is built — correctly, per this
repo's zero-cost/local-only scope — but it's named explicitly rather
than left implicit.

## Incident Runbook

Full runbook: [`docs/incident-runbook.md`](docs/incident-runbook.md).
Five scenarios, each mapped to a real alert rule in
`prometheus/alert-rules.yml` or a finding directly demonstrated in
`docs/threat-model.md` — not generic filler.

| Scenario | Trigger | Coverage |
|---|---|---|
| Prometheus killed via unauthenticated `/-/quit` | Threat model finding | ⚠️ Detection gap: indistinguishable in logs from a normal restart |
| Fake alert injected via Alertmanager's unauthenticated API | Threat model finding | Detectable — compare Prometheus's own rule state against Alertmanager's active alerts |
| Collector pipeline stalled | Golden Signals dashboard goes flat | `TargetDown`, Pipeline Health panels |
| High error rate / latency | Dashboard + trace correlation | `HighErrorRate`, `HighLatencyP99` |
| Event loop saturation | Dashboard | `EventLoopSaturation` |

The two security-driven scenarios have **no alert coverage today** —
stated plainly in the runbook itself, matching the threat model's
own accepted-risk framing rather than implying it's handled.

## Documentation

Additional documentation is available under the `docs/` directory.
Architecture decisions are recorded using ADRs in `docs/adr/`.

## Contributing

Please read `CONTRIBUTING.md` before submitting changes.

## License

This project is licensed under the MIT License.
