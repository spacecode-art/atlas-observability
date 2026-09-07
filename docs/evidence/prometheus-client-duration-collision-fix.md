# Evidence: Prometheus HELP-string Collision — instrumentation-http vs instrumentation-undici

Date: 2026-09-04
Related: ADR-0004

## Before fix — recurring error on every scrape

```
atlas-otel-collector  | 2026-09-04T10:37:43.021Z  error  prometheusexporter@v0.106.0/log.go:23
  error gathering metrics: collected metric http_client_request_duration_seconds
  label:{name:"http_request_method" value:"GET"} label:{name:"http_response_status_code" value:"200"}
  label:{name:"job" value:"atlas-demo-apib"} label:{name:"server_address" value:"xecomdfrdxbjccnldduv.supabase.co"}
  ...
  has help "Measures the duration of outbound HTTP requests."
  but should have "Duration of HTTP client requests."
```

Recurred on every ~15s scrape cycle, timestamps 10:37:43 through 10:42:13 observed continuously.

## Source-level confirmation of the two conflicting descriptions

```
instrumentation-http/build/src/http.js:43:
  description: 'Duration of HTTP client requests.'

instrumentation-undici/build/src/undici.js:58:
  description: 'Measures the duration of outbound HTTP requests.'
```

Versions confirmed against `package-lock.json`: `@opentelemetry/instrumentation-http@0.222.0`, `@opentelemetry/instrumentation-undici@0.32.0`.

## After fix — clean scrape, correctly deduplicated series

```
$ docker compose logs otel-collector --since=30s | grep -i "error gathering"
(no output)

$ curl -s 'http://localhost:9090/api/v1/query?query=http_client_request_duration_seconds_count'
{
  "metric": {
    "__name__": "http_client_request_duration_seconds_count",
    "exported_job": "atlas-demo-apib",
    "http_request_method": "GET",
    "http_response_status_code": "200",
    "server_address": "xecomdfrdxbjccnldduv.supabase.co",
    "server_port": "443",
    "url_scheme": "https"
  },
  "value": [1788518920.087, "3"]
}

$ curl -s 'http://localhost:9090/api/v1/query?query=http_server_request_duration_seconds_count'
# 3 series: atlas-demo-api/200 (4), atlas-demo-apib/200 (2), atlas-demo-api/304 (83)
```

Single series for `http_client_request_duration_seconds`, sourced from `instrumentation-undici` only, no collision errors, `http_server_request_duration_seconds` unaffected throughout (see note in ADR-0004 — this metric was never actually broken by this bug; its earlier emptiness had a separate cause).

## Note on `exported_job: "atlas-demo-apib"`

Both metrics above show the known trailing-`b` bug (`docs/adr` / earlier ledger: env-runner worker process's `service.name` reads `atlas-demo-apib`) independently confirmed again here via the `EnvDetector` resource in `OTEL_DEBUG=1` output:

    EnvDetector found resource. ResourceImpl {
      _rawAttributes: [ [ 'service.name', 'atlas-demo-apib' ] ], ...
    }

process.command: `node_modules/env-runner/dist/runners/node-worker/worker.mjs`

Still unfixed; still tracked against the same next step already in the ledger (`grep -rn OTEL_SERVICE_NAME node_modules/env-runner/dist/runners/vercel/worker.mjs` — note the path differs slightly here: `node-worker`, not `vercel`, worth re-checking that grep target against the `node-server` preset's actual worker path when this is picked up).