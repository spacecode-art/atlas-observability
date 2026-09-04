# Threat Model — Atlas Observability

STRIDE analysis of this repo's actual running configuration, not
generic boilerplate. Every finding below is either read directly from
committed config or was empirically demonstrated during this repo's
own build sessions — cited inline.

## Scope

This stack is designed to run on `localhost` for zero-cost portfolio
purposes. Every finding here assumes the realistic failure mode: this
exact `docker-compose.yml`, unmodified, deployed on a shared network
or a cloud VM with a public IP and no additional firewall — which is
exactly the kind of misconfiguration that turns a safe local demo
into a real incident. That's the threat surface worth documenting;
threat-modeling `localhost`-only exposure against a threat actor who
already has local shell access adds nothing.

## Asset Inventory

| Component | Port | Binding | Auth |
|---|---|---|---|
| OTel Collector — OTLP receivers | 4317 (gRPC), 4318 (HTTP) | `0.0.0.0`, published to host | None |
| OTel Collector — self-telemetry | 8888 | `0.0.0.0`, published to host | None |
| Prometheus | 9090 | published to host | None, plus `--web.enable-lifecycle` |
| Loki | 3100 | published to host | None |
| Tempo | 3200 | published to host | None |
| Alertmanager | 9093 | published to host | None |
| Grafana | 3000 | published to host | Anonymous **Viewer** role enabled |

Every service in this stack is unauthenticated at the network layer.
This is a deliberate, documented tradeoff for local demo purposes —
not an oversight — but it's the finding that matters most, so it's
stated plainly rather than buried under per-component detail below.

---

## STRIDE Analysis

### Spoofing

**OTLP ingest has no client authentication.** `otel-collector/config.yaml`
defines the `otlp` receiver with no `auth` extension configured. Any
process that can reach `4317`/`4318` can push spans, logs, or metrics
claiming to be `atlas-demo-api` — there's nothing to prevent a
different process from spoofing that service name. Combined with the
already-confirmed `atlas-demo-apib` bug (real telemetry can already
carry an unintended service name due to a code bug, not an attacker),
this shows the pipeline has no mechanism to distinguish "wrong service
name from a bug" from "wrong service name from a spoofed client" —
they'd look identical downstream.

- **Mitigation status**: accepted risk for local-only deployment.
  Real fix if ever exposed: `otlp` receiver `auth` extension
  (e.g. bearer token) — not built, out of scope for zero-cost local
  demo per the roadmap.

### Tampering

**Alertmanager's write API is unauthenticated** — demonstrated
directly this session, not theoretical: synthetic alerts were posted
to `POST /api/v2/alerts` with a bare `curl`, no credentials, and were
accepted and routed to Slack. Anyone with network access to `9093`
can inject arbitrary fake alerts (alert-fatigue / crying-wolf attack
against the on-call Slack channel) or silence real ones via the same
unauthenticated API.

- **Mitigation status**: accepted risk locally. If exposed, requires
  a reverse proxy with auth in front of `9093` — Alertmanager itself
  has no native auth.

### Repudiation

**No audit log of who changed what.** Grafana anonymous Viewer access
means dashboard *views* aren't attributable to a user (by design —
Viewer role can't edit). More relevant: Prometheus's `/-/reload` and
Alertmanager's config are read from disk with no change-tracking
beyond git history on the host machine. Acceptable for a single-operator
local demo; would not be acceptable multi-tenant.

- **Mitigation status**: out of scope — git history is the audit
  trail for this deployment model, which is sufficient for its actual
  use case.

### Information Disclosure

**Confirmed this session**: a real Slack webhook URL was pasted in
plaintext into this chat session during Alertmanager setup. That URL
was treated as burned — the fix applied was routing the secret
through `slack_configs.api_url_file` (a gitignored file, never
committed) going forward, and the operator was told to rotate the
webhook in Slack's app settings. This is real evidence of *why* the
`api_url_file` pattern matters, not a hypothetical example.

**Grafana anonymous Viewer** can use Explore to run arbitrary
PromQL/LogQL against Prometheus and Loki directly — not just the
provisioned dashboard queries. The `attributes/scrub` processor in
the Collector (deletes `net.peer.name`, `http.client_ip`) is the
actual control here: anonymization happens before data reaches
Prometheus/Loki, so an anonymous viewer running arbitrary queries
still can't see identifying data that was never stored. This is
belt-and-suspenders working as designed (see ADR-0001) — worth
naming here as a real mitigation, not just a coincidence.

- **Mitigation status**: partially mitigated (scrub processor).
  Residual: Grafana anonymous access itself is unmitigated if this
  stack were ever exposed beyond localhost.

### Denial of Service

**Confirmed via source, not assumed**: Prometheus is run with
`--web.enable-lifecycle`, which was added specifically so
`docker compose restart prometheus` picks up config changes without a
full container recreate. Checked the actual v2.55.1 source
(`web/web.go`) — this flag doesn't just enable `/-/reload`, it also
enables an **unauthenticated `POST /-/quit`**, which shuts the
process down. Anyone with network access to `9090` can kill
Prometheus with one `curl` command.

**Unauthenticated OTLP ingest** (see Spoofing) also means anyone with
network access can flood the Collector with junk telemetry, exhausting
its `batch` processor queue or downstream storage (Loki/Tempo disk).

- **Mitigation status**: accepted risk for local-only deployment.
  Real fix if ever exposed: don't use `--web.enable-lifecycle` on a
  publicly-reachable instance, or put it behind an auth-enforcing
  reverse proxy — Prometheus has no way to scope the lifecycle
  endpoints down without one.

### Elevation of Privilege

Not applicable in the traditional sense — there's no user/role model
in this stack beyond Grafana's anonymous-Viewer-vs-authenticated-admin
split, and the admin password is already handled correctly
(`GF_SECURITY_ADMIN_PASSWORD` sourced from a required `.env` var, not
hardcoded — see `docker-compose.yml`). No privilege boundary exists
for Prometheus, Loki, Tempo, Alertmanager, or the Collector to
escalate within.

---

## Summary — What Would Actually Need to Change Before Public Exposure

1. Auth extension on the OTLP receiver (Spoofing, DoS)
2. Reverse proxy with auth in front of Alertmanager, Prometheus, Loki,
   Tempo, and Collector self-telemetry (Tampering, DoS, Information
   Disclosure)
3. Drop `--web.enable-lifecycle` or gate it behind the same proxy
   (Denial of Service)
4. Disable Grafana anonymous access, or scope Explore away from the
   anonymous Viewer role (Information Disclosure)

None of this is built, and per the zero-cost/local-only scope of this
repo, none of it needs to be — but a threat model that didn't name
these clearly would just be theater. This one names the real, verified
gaps instead.