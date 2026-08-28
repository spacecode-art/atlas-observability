# Contributing

Thank you for your interest in contributing to Atlas Observability.

## Development Workflow

1. Create a feature branch from `main`.
2. Make focused, well-documented changes.
3. Run `docker compose up -d` and verify all services report healthy
   before committing changes to any stack config (`docker-compose.yml`,
   `otel-collector/`, `prometheus/`, `loki/`, `tempo/`,
   `alertmanager/`) — see `docs/evidence/stack-health-check.md` for
   the expected readiness checks per service.
4. Open a Pull Request with a clear description.

## Anonymization Requirement

Per [ADR-0001](docs/adr/ADR-0001-use-tawira-instead-of-sample-app.md),
telemetry in this repo comes from a real application (Tawira) with
identifying details genericized. Any change touching instrumentation
code, OTel SDK configuration, or the Collector's `attributes/scrub`
processor must not introduce real service names, route names, or
other identifying details into dashboards, evidence files, or
committed screenshots. If a change adds a new identifying field, add
it to the scrub processor in the same PR.

## Commit Messages

Follow a clear, descriptive style.

Examples:

```text
feat(otel): instrument Tawira API layer with OTel SDK
docs: record ADR-0003 Golden Signals dashboard design
fix(loki): correct retention period in loki-config.yaml
```

## Code Standards

- Write clear documentation.
- Keep stack configs reusable and self-hosted — no managed/paid
  observability services introduced without a cited ADR explaining
  the cost tradeoff.
- Prefer small, focused commits.
- Avoid committing secrets, `.env` files, or real Tawira identifying
  details.