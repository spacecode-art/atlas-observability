# ADR-0001: Use Tawira Instead of a Sample App

## Status
Accepted

## Context
The original Atlas plan called for OpenTelemetry instrumentation on
"a small sample app." `atlas-security` already faced this same choice
for its SBOM/Grype/Cosign pipeline and chose Tawira, a real production
SaaS application, over a throwaway sample app — see `atlas-security`
ADR-0001. The reasoning transfers directly to observability: a real,
evolving codebase with real request patterns produces real, evolving
telemetry, where a static sample app's traffic shape would be
artificial and stay that way.

Tawira is a multi-tenant inventory/POS SaaS application (TanStack
Start, Node/Nitro runtime, Supabase-backed) that I run in production.
It is a private repository, which introduces two real constraints a
sample app would not have:

1. Tawira's service/route names and business logic are not meant to
   be public.
2. Running instrumentation experiments against a live system with
   real users carries real operational risk.

## Decision
Instrument Tawira, but:
- Run it locally against a local Supabase instance (`supabase start`),
  never against production data, for all `atlas-observability` work.
- Anonymize all identifying details — service names, route names —
  to a generic `atlas-demo-*` convention, enforced at two layers: the
  OTel SDK configuration in Tawira itself, and a second attribute-scrub
  processor in the OTel Collector (see `otel-collector/config.yaml`)
  as defense in depth against a misconfigured SDK leaking real names.
- State this plainly in the README rather than presenting `atlas-demo-*`
  as if it were the application's real identity.

## Consequences
- Telemetry patterns (latency distributions, error rates, trace
  shapes) are real and representative of production Tawira, even
  though the labels are not.
- No production ingress or credentials are ever required by this
  repo — the entire stack, including the instrumented app, runs
  locally via Docker Compose.
- A reviewer cannot verify claims about Tawira's real production
  behavior directly, since Tawira itself is private. This is an
  accepted tradeoff, consistent with `atlas-security`'s same choice.