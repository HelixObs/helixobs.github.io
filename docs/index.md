<div style="background:#1a2335; border-radius:12px; padding:24px 32px; margin-bottom:32px;">
  <img src="assets/wordmark.svg" alt="HelixObs" style="width:100%; max-width:480px; display:block;" />
</div>

**Entity-centric observability for scientific instrument pipelines.**

## What is an Entity?

An **entity** is any data product your pipeline creates, transforms, or consumes — a raw observation, a detection candidate, a calibration solution, a science file. It has a stable string ID that you choose (a database key, a filename hash, a UUID) and that ID follows the product across every processing stage, compute node, and instrument that touches it.

Every entity accumulates:

- a **provenance graph** — which upstream entities it was derived from, and which downstream entities it produced
- a **trace** — the distributed [OpenTelemetry](https://opentelemetry.io/) trace of every processing stage
- **logs** — all log lines emitted while any stage was active, correlated by entity ID
- **events** — named domain milestones and errors attached at any stage

This is the core idea: instead of asking *"what did service X do?"*, you ask *"what happened to entity Y, across every service that touched it?"*

## Why HelixObs?

Standard observability tools — Datadog, Prometheus, OpenTelemetry — were designed for web services. They answer operational questions: is the service up, is latency acceptable, what is the error rate? They have no concept of domain data products, and no ability to track a specific item of data across multiple disjoint asynchronous processes.

Instrument and data pipelines are fundamentally different. A single result may be the product of hundreds of parallel processing branches, aggregated over minutes or hours, across dozens of hosts. **Failures in these pipelines have real consequence.** A processing failure that goes undetected for a day means that data window is gone — it cannot be reprocessed, and its scientific or operational value is permanently lost.

Three specific problems motivated HelixObs:

**1. Silent failures with delayed consequence.** In a web service, a failed request surfaces immediately. In a data pipeline, failures are often discovered days later — when someone notices results look wrong. By then the cause is difficult to trace and the impact is hard to quantify. Standard alerting on CPU and error rates does not catch the cases that matter: a job that completes successfully but produces wrong output, or a stage that silently drops data under load.

**2. Provenance is a DAG, not a tree.** Standard distributed tracing assumes one parent, many children — a synchronous request tree. Data pipelines produce directed acyclic graphs: N upstream entities are combined into one result, which fans out to M downstream processes. No existing tracing tool can represent this causal structure, track a data product through it, or answer "which upstream inputs contributed to this output?"

**3. Existing tools leave a gap.** Log aggregators give you searchable text. Distributed tracing gives you request waterfalls. Infrastructure monitoring gives you CPU and memory. None of them give you a unified view of what happened to a specific data product — across every process, every host, every stage — in one place. Teams fill this gap with custom scripts and dashboards that accumulate technical debt and are never quite trusted.

HelixObs is a production implementation of entity-centric observability that closes this gap, built on OpenTelemetry so it works alongside the tools you already have.

## What you get

| | |
|---|---|
| **Provenance graph** | Full DAG of how each entity was produced — queryable via the Grafana Entity Inspector |
| **Correlated logs** | Every log line emitted while processing an entity carries its ID and trace ID — search across stages in one query |
| **Event timeline** | Named domain events (`helix.event.*`) and errors (`helix.error`) attached to entities and surfaced in Grafana |
| **Notifications** | Slack messages and GitHub issues opened automatically for errors, with dedup and rate limiting |
| **AI troubleshooting** | Sherlock investigates entity errors on demand: fetches logs, traces, provenance, and source code, then classifies the root cause |

## How it works

```
Instrument pipeline                 HelixObs stack
─────────────────                   ──────────────────────────────────
helixobs client library
  create() / operate()
  ─► BatchSpanProcessor   ─────────► Herald :4317 (gRPC)
                                       │  enrich spans
                                       │  resolve parent links
                                       │  write to TimescaleDB
                                       │  emit Prometheus metrics
                                       └─► OTel Collector :4317
                                             │
                                             ├─► Tempo  (traces)
                                             └─► Loki   (OTLP logs)

  configure_logging()
  ─► stdout JSON          ─────────► Alloy (Docker scrape)
                                       └─► Loki (sidecar logs)

                                     Prometheus (scrapes herald,
                                       sherlock, otel-collector…)

                                     Grafana
                                       datasources: Loki, Tempo,
                                         Prometheus, TimescaleDB
```

### The Herald

The **herald** is HelixObs's central intelligence layer — the only HelixObs-specific service an instrument pipeline talks to directly. It listens for OTLP spans on port `4317`, the standard OpenTelemetry port, so no custom protocol is required.

When a span arrives carrying `helix.entity.id`, the herald does the work that standard OTel cannot: it resolves parent IDs across process boundaries, writes the entity provenance graph to TimescaleDB, dispatches error notifications to Slack and GitHub, and forwards the enriched span batch onward to the standard OTel Collector. Spans without `helix.entity.id` are forwarded unchanged — the herald is fully transparent to non-HelixObs traffic.

From a pipeline team's perspective, the herald is a single endpoint to configure and forget. The `helixobs` client library handles the connection.

## Documentation structure

| | |
|---|---|
| [Platform Tour](tour.md) | Screenshots of every UI view — what you get after instrumenting |
| [Getting Started](getting-started.md) | Instrument your first pipeline in 5 minutes |
| [For Scientists & Developers](client/index.md) | Client library reference, logging modes, auth |
| [For Operators](operator/index.md) | Stack deployment, Alloy config, dashboards, notifications |
