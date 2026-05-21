# Grafana Dashboards

HelixObs ships five Grafana dashboards, provisioned automatically from JSON files in `deploy/grafana/dashboards/`. Changes to those files take effect immediately in a running stack — no restart required.

---

## Entity Inspector

**Purpose:** Deep-dive into a single entity — its provenance graph, processing trace, events timeline, and correlated logs.

**Data sources:** TimescaleDB (provenance graph), Tempo (trace), Loki (logs)

**How to use:** Enter an entity ID in the top variable selector. The dashboard fetches the entity's trace ID from TimescaleDB, then pulls the full trace from Tempo and all correlated logs from Loki.

**Log queries used:**

```logql
{helix_instrument_id=~".+"} | helix_entity_id = `<entity-id>`
{helix_instrument_id=~".+"} | trace_id = `<trace-id>`
```

!!! warning "Requires correct Alloy pipeline for sidecar logs"
    These queries rely on `helix_entity_id` and `trace_id` being present as Loki structured metadata. For instruments using `otlp=False`, the Alloy pipeline must include `stage.structured_metadata` as described in [Log Collection](log-collection.md). Without it, the logs panel will be empty even though logs exist in Loki.

---

## Pipeline Process Logs

**Purpose:** Tail and filter logs from a named pipeline process in real time. Useful for debugging a specific stage without navigating individual entities.

**Data source:** Loki

**How to use:** Select a process from the `helix_process_name` dropdown. Filter by log level. The process list is populated from the `helix_process_name` stream label in Loki.

**Log queries used:**

```logql
{helix_process_name="<process-name>"} | level=~"<level>"
```

!!! warning "Requires helix_process_name as a stream label"
    For instruments using `otlp=False`, `helix_process_name` must be promoted in the Alloy `stage.labels` block. Without it the process dropdown will be empty.

    Instrument teams must also set `process_name` in `setup()` using the `INST_ID/pipeline/stage` naming convention — see [Structured Logging](../client/logging.md#process-naming).

---

## Error Entities

**Purpose:** Table of all entities that have recorded a `helix.error` event, grouped by instrument. Entry point for triage.

**Data source:** Prometheus, TimescaleDB

**Dependencies:** No Loki or Alloy configuration required — data comes entirely from herald metrics and TimescaleDB.

---

## Platform Health

**Purpose:** Infrastructure health of the HelixObs stack itself — herald throughput, parent resolution rate, DB write latency, trace store size, OTel Collector pipeline metrics, Loki and Tempo ingestion rates, host CPU/memory/disk.

**Data source:** Prometheus (all metrics)

**Dependencies:** Prometheus scrape targets must be reachable. No Loki or Alloy configuration required.

---

## Sherlock Cost

**Purpose:** Token usage, cost, query latency, and tool call breakdown for the Sherlock AI troubleshooting service.

**Data source:** Prometheus

**Dependencies:** Sherlock must be running and `ANTHROPIC_API_KEY` must be set. No Loki or Alloy configuration required.

---

## Dashboard dependency summary

| Dashboard | Loki | Alloy pipeline required | Tempo | Prometheus | TimescaleDB |
|---|---|---|---|---|---|
| Entity Inspector | Yes | If using sidecar path | Yes | — | Yes |
| Pipeline Process Logs | Yes | If using sidecar path | — | — | — |
| Error Entities | — | — | — | Yes | Yes |
| Platform Health | — | — | — | Yes | — |
| Sherlock Cost | — | — | — | Yes | — |
