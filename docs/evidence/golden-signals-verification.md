# Golden Signals Dashboard — Metric Verification, 2026-09-02

Metric names verified against actual npm tarballs for the pinned
OTel versions before being written into dashboard queries — not
assumed from memory or older blog posts. See commit
`feat(dashboards): add Golden Signals dashboard for Tawira`.

## HTTP request metric (rate / errors / latency)

```
$ curl -s http://localhost:9090/api/v1/label/__name__/values
http_server_request_duration_seconds_bucket
http_server_request_duration_seconds_count
http_server_request_duration_seconds_sum
```

Confirmed via `@opentelemetry/instrumentation-http@0.222.0` source
(`http.js`): the histogram is registered under
`METRIC_HTTP_SERVER_REQUEST_DURATION`, exported by the Collector's
Prometheus exporter with the standard `s` → `_seconds` unit suffix.

## Saturation metric — found, not assumed missing

Initially assumed no saturation signal existed, since
`instrumentation.mjs` never imports `@opentelemetry/host-metrics`.
Wrong — `@opentelemetry/auto-instrumentations-node@0.80.0` bundles
`@opentelemetry/instrumentation-runtime-node@0.35.0` automatically,
which emits event-loop and V8 heap metrics with zero explicit wiring:

```
$ curl -sG http://localhost:9090/api/v1/query \
    --data-urlencode 'query=nodejs_eventloop_utilization_ratio'

"metric": {"exported_job": "atlas-demo-api"},  "value": [.., "0.0101"]
"metric": {"exported_job": "atlas-demo-apib"}, "value": [.., "0.0193"]
```

Note: the `atlas-demo-apib` trailing-`b` (known bug, see Current
Status) shows up here too — it's not limited to the Tempo trace
resource attribute originally logged; it pollutes the metrics
`exported_job` label as well. Same root cause, wider blast radius
than first scoped.

## Collector pipeline health (self-telemetry)

Port 8888 was exposed to the host in an earlier session but never
actually scraped by Prometheus — confirmed missing from `/targets`,
fixed by adding the `otel-collector-self` job, confirmed `health: up`
after a Prometheus restart (config is read at container start, not
hot-reloaded just because the mounted file changed):

```
$ curl -s http://localhost:9090/api/v1/query?query=otelcol_receiver_accepted_spans
"job": "otel-collector-self", "value": [.., "978"]
```

**Result:** all six dashboard panels (request rate, error rate,
p50/p95/p99 latency, status code breakdown, event-loop saturation,
collector accept-rate) confirmed rendering with live data in Grafana
at `http://localhost:3000`.