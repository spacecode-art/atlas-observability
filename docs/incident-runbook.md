# Incident Runbook — Atlas Observability

Built directly from this repo's actual alert rules (`prometheus/alert-rules.yml`)
and threat model (`docs/threat-model.md`) — every scenario below maps to
something that either fires a real alert in this stack or was directly
demonstrated during this repo's build. No generic filler scenarios.

## Severity Quick Reference

| Alert | Fires on | Slack receiver |
|---|---|---|
| `HighErrorRate` | 5xx ratio > 5% for 5m | `slack-critical` |
| `TargetDown` | any scrape target unreachable 2m | `slack-critical` |
| `HighLatencyP99` | p99 latency > 2s for 5m | `slack-warning` |
| `EventLoopSaturation` | event-loop utilization > 0.9 for 5m | `slack-warning` |

**Known gap** (see threat model): the two security-driven scenarios
below — unauthenticated `/-/quit` and fake alert injection — have
**no alert coverage today**. Both are metric-threshold-blind by
nature (a killed Prometheus can't alert on its own death beyond
`TargetDown` firing from Alertmanager's perspective, and injected
fake alerts look identical to real ones to Alertmanager). Tracked
honestly as a follow-up, not silently assumed covered.

---

## Scenario 1: Prometheus Unreachable / Killed

**Symptom**: `TargetDown` fires in Slack, or Grafana dashboards go
blank/show "no data", or `http://localhost:9090` stops responding
entirely.

**Detection**:
```bash
curl -s http://localhost:9090/-/healthy
docker compose ps prometheus
docker compose logs prometheus --tail 50
```

**Root cause branches**:
- Container crashed/OOM → `docker compose logs prometheus` shows the
  panic/OOM kill; check `docker stats` for memory pressure.
- **Process was killed via the unauthenticated `POST /-/quit`
  endpoint** (confirmed exposed in the threat model — this flag,
  `--web.enable-lifecycle`, is required for our own config-reload
  workflow, so it can't simply be removed without breaking the
  restart-to-reload pattern used throughout this repo's sessions).
  If the container shows a clean exit (not OOM, not a panic) and
  restarted itself via Compose's restart policy, this is the likely
  cause — Compose logs alone won't distinguish "someone called
  `/-/quit`" from "someone ran `docker compose restart`", since both
  produce a clean shutdown. This is itself a real gap: **we cannot
  currently tell those two apart from logs alone.**

**Mitigation**:
```bash
docker compose restart prometheus
sleep 5
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep health
```

**Follow-up if `/-/quit` is suspected** (not proven — see gap above):
this stack is local-only by design, so the realistic trigger is
another local process or a port left open on a shared network, not a
remote attacker. Confirm nothing else on the host network can reach
`9090` before treating it as a false alarm.

---

## Scenario 2: Fake/Injected Alerts in Slack

**Symptom**: an alert fires in `#atlas-alerts` that doesn't correspond
to anything visible in Grafana — e.g. `HighErrorRate` firing while the
Golden Signals dashboard shows a normal, low error rate.

**Detection** — check whether Alertmanager's view of the alert is
backed by a real Prometheus rule evaluation or was injected directly:
```bash
# Does Prometheus itself think this rule is firing?
curl -s http://localhost:9090/api/v1/rules | python3 -m json.tool | grep -A5 "HighErrorRate"

# Compare against what Alertmanager has active
curl -s http://localhost:9093/api/v2/alerts | python3 -m json.tool
```
If Prometheus's own rule evaluation shows `state: inactive` while
Alertmanager shows the same alert as `active`, it did **not** come
from a real rule firing — it was posted directly to Alertmanager's
API, exactly the same way this repo's own synthetic-alert verification
did (`POST /api/v2/alerts`, no auth required — confirmed in the threat
model).

**Mitigation** — silence and remove the injected alert:
```bash
# resolve it directly (same mechanism used to clean up our own synthetic alerts)
curl -H "Content-Type: application/json" -d '[{
  "labels": {"alertname": "<name>", "severity": "<severity>"},
  "annotations": {"description": "resolved - confirmed injected, not a real rule firing"},
  "startsAt": "'"$(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S.000Z)"'",
  "endsAt": "'"$(date -u +%Y-%m-%dT%H:%M:%S.000Z)"'"
}]' http://localhost:9093/api/v2/alerts
```

**This is an accepted risk for local-only deployment** (see threat
model) — there is no fix within Alertmanager itself; only a reverse
proxy with auth in front of `9093` closes this.

---

## Scenario 3: Collector Pipeline Stalled

**Symptom**: Golden Signals dashboard panels for Tawira go flat (no
new data points), but the app itself appears to be running.

**Detection** — this is exactly what the Pipeline Health row exists
for:
```bash
curl -sG http://localhost:9090/api/v1/query \
  --data-urlencode 'query=rate(otelcol_receiver_accepted_spans[5m])'
```
A value near zero while Tawira is actively serving requests means the
Collector isn't receiving or isn't accepting telemetry.

**Root cause branches**:
- Collector container down → `docker compose ps otel-collector`
- Tawira can't reach the Collector (wrong `OTEL_EXPORTER_OTLP_ENDPOINT`,
  Docker network issue if running Tawira in the same Compose network)
  → check `instrumentation.mjs` config and confirm Tawira's process
  logs for OTLP export errors
- Collector receiving but not exporting (e.g. Loki/Tempo/Prometheus
  exporter target down) → check `docker compose logs otel-collector`
  for exporter-side errors specifically, not just receiver-side

**Mitigation**:
```bash
docker compose restart otel-collector
# re-verify
curl -sG http://localhost:9090/api/v1/query --data-urlencode 'query=up{job="otel-collector-self"}'
```

---

## Scenario 4: High Error Rate / High Latency (Tawira)

**Symptom**: `HighErrorRate` or `HighLatencyP99` fires.

**Detection**:
```bash
# breakdown by status code, same query as the dashboard's status-code panel
curl -sG http://localhost:9090/api/v1/query \
  --data-urlencode 'query=sum(rate(http_server_request_duration_seconds_count[5m])) by (http_response_status_code)'
```
Cross-reference with Tempo (Grafana Explore → Tempo) for the specific
failing traces — the nested-span structure confirmed working in
ADR-0003's investigation (SSR span → outgoing client span → worker
process span) is exactly what makes root-causing a specific slow/failed
request possible here, not just aggregate metrics.

**Mitigation**: application-specific — this runbook covers detection
and diagnosis, not a fix, since the fix depends entirely on what the
trace reveals (DB query, external call, code bug).

---

## Scenario 5: Event Loop Saturation

**Symptom**: `EventLoopSaturation` fires (`nodejs_eventloop_utilization_ratio > 0.9`).

**Detection**:
```bash
curl -sG http://localhost:9090/api/v1/query --data-urlencode 'query=nodejs_eventloop_delay_p99_seconds'
curl -sG http://localhost:9090/api/v1/query --data-urlencode 'query=sum(v8js_memory_heap_used_bytes)'
```
Rising heap alongside rising event-loop delay suggests memory pressure
causing GC pauses, not just raw request volume — check both together,
not `EventLoopSaturation` in isolation.

**Note**: watch for the `atlas-demo-apib` labeling bug (known,
unfixed) when reading these — the worker process reports under a
different `exported_job` label than the main process, so saturation
on the worker won't show up under `atlas-demo-api` queries filtered
by that label.

---

## Postmortem Trigger

Any `critical`-severity alert (`HighErrorRate`, `TargetDown`) that
took longer than 15 minutes to resolve, or any confirmed security
finding from the threat model observed in the wild (not just a
synthetic test), gets a written postmortem. None have occurred yet —
this repo's only "incidents" to date were the deliberate synthetic
tests used to prove the alerting pipeline itself works.