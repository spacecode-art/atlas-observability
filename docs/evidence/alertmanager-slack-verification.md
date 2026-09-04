# Alertmanager → Slack Routing — End-to-End Verification, 2026-09-04

Config validity alone (`promtool check rules`, `amtool check-config`)
doesn't prove delivery — a receiver can match correctly in the API
response while the actual Slack POST silently fails. Verified the
full path instead.

## Static validation

Both prebuilt images set the binary itself as `ENTRYPOINT`, so
`promtool`/`amtool` as a bare `CMD` argument fails with
`unexpected promtool` — not a config error, a Docker invocation one.
Fixed with `--entrypoint`:

```
$ docker run --rm --entrypoint promtool -v "$(pwd)/prometheus/alert-rules.yml:/rules.yml:ro" \
    prom/prometheus:v2.55.1 check rules /rules.yml
SUCCESS: 4 rules found

$ docker run --rm --entrypoint amtool -v "$(pwd)/alertmanager/alertmanager.yml:/alertmanager.yml:ro" \
    prom/alertmanager:v0.27.0 check-config /alertmanager.yml
SUCCESS — 3 receivers, 0 templates
```

## Routing proof (synthetic alerts, not real incidents)

```
$ curl -d '[{"labels":{"alertname":"HighErrorRate","severity":"critical"},...}]' \
    http://localhost:9093/api/v2/alerts
# GET /api/v2/alerts confirms: "receivers": [{"name": "slack-critical"}]

$ curl -d '[{"labels":{"alertname":"EventLoopSaturation","severity":"warning"},...}]' \
    http://localhost:9093/api/v2/alerts
# GET /api/v2/alerts confirms: "receivers": [{"name": "slack-warning"}]
```

Both `severity="critical"`/`severity="warning"` matchers route
independently and correctly — the second route in the list isn't
just falling through to the first.

## Delivery proof (the part the API can't confirm)

Both messages confirmed delivered and correctly formatted in the
`#atlas-alerts` Slack channel:

- `🚨 [CRITICAL] HighErrorRate` — synthetic test alert
- `⚠️ [WARNING] EventLoopSaturation` — synthetic warning test

Screenshot: `docs/evidence/screenshots/alertmanager-slack-delivery.png`

Both synthetic alerts resolved immediately after (posted with
`endsAt` in the past), confirmed via `GET /api/v2/alerts` returning
`[]` — no false-firing alert left lingering.

## Secret handling

Webhook URL is never inline in committed YAML — routed through
`slack_configs.api_url_file`, confirmed supported in Alertmanager
v0.27.0 via the actual tagged source
(`config/notifiers.go`, `SlackConfig.APIURLFile`), not assumed from
older docs. Real URL lives at `alertmanager/secrets/slack_webhook_url`
(gitignored); `.example` committed to document the pattern.