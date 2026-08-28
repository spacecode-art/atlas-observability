# ADR-0002: Loki and Tempo Ring-Readiness Startup Race

## Status
Accepted

## Context
On first `docker compose up`, `curl /ready` against both Loki and
Tempo returned `503` while `docker compose ps` reported both
containers as `Up`. A container process running is not the same as
the service inside it being ready to serve traffic — this gap is
exactly why health is verified with `curl` against each service's
readiness endpoint, not just `docker compose ps`, in this repo's
deployment checks (see `docs/evidence/stack-health-check.md`).

Investigation via `docker compose logs` showed both services logged
successful startup (`"Loki started"`, `"Tempo started"`) and their
internal single-node "ring" (used for coordination even with one
instance) reaching `ACTIVE` state within roughly 150ms–5s of process
start. Re-running the same `curl` checks ~1 minute later returned
`200`/`ready` for both. This is a known, non-fatal startup
characteristic of Loki and Tempo's ring-based architecture, not a
misconfiguration in this repo's `loki-config.yaml` or
`tempo-config.yaml`.

A separate, unrelated log line was also observed in Loki at
`error` level: `msg="failed to get token ranges for ingester"
err="zone not set"`. This is a known non-fatal warning in single-binary
Loki deployments without zone-awareness configured, irrelevant for a
single-node local stack. Documented here rather than silently ignored,
per this project's standard of tracking known issues explicitly
instead of letting them go unrecorded because nothing crashed.

## Decision
Accept the readiness race as expected behavior. No config change is
needed. When verifying stack health (locally, in CI, or before
capturing evidence), allow a short delay (15–30s) after
`docker compose up` before checking `/ready` endpoints, rather than
checking immediately after `docker compose ps` shows `Up`.

## Consequences
- Any future CI health-check job for this repo must include a wait
  step before asserting readiness, or it will produce false-negative
  failures on Loki/Tempo specifically.
- The `"zone not set"` error-level log line is expected noise in this
  single-node setup and should not be mistaken for a real failure
  during future debugging.