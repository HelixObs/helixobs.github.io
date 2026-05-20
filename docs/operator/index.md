# For Operators

This section covers everything needed to deploy and operate the HelixObs stack for one or more instruments.

## Pages in this section

| | |
|---|---|
| [Stack Architecture](architecture.md) | All services, data flows, and ports |
| [Deployment](deployment.md) | Docker Compose setup, environment variables, volumes |
| [Log Collection](log-collection.md) | Alloy vs OTLP — what each requires and what breaks without it |
| [Grafana Dashboards](dashboards.md) | What each dashboard shows and what data it depends on |
| [Notifications](notifications.md) | Slack and GitHub Issues — config format, rate limiting, silence rules |
| [Authentication Setup](auth.md) | JWT auth backends, key rotation, rollout |
| [Adding an Instrument](new-instrument.md) | Instrument YAML config, auth setup, onboarding checklist |

## Operator responsibilities at a glance

1. **Deploy the stack** — gateway, OTel Collector, TimescaleDB, Loki, Tempo, Prometheus, Grafana, Alloy
2. **Configure log collection** — decide per-instrument whether Alloy or OTLP delivers logs, and ensure the required pipeline is in place
3. **Configure instrument YAML** — one file per instrument for notifications and auth
4. **Provide connection details to instrument teams** — gateway address, auth endpoint, credential
5. **Monitor** — Platform Health dashboard, error rate alerts, Sherlock for AI-assisted diagnosis
