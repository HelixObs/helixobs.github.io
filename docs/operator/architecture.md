# Stack Architecture

## Services

The HelixObs stack is composed of off-the-shelf open-source components plus the HelixObs herald and Sherlock.

```
Instrument pipeline
  │  OTLP gRPC (traces)
  ▼
┌─────────────────────────────────────────────────────────┐
│  Herald  :4317                                         │
│  ─ enriches spans with provenance links                 │
│  ─ writes entities / events / operations → TimescaleDB  │
│  ─ forwards enriched spans → OTel Collector             │
│  ─ emits Prometheus metrics  :2112                      │
│  ─ serves HTTP query API     :8080                      │
└───────────────┬─────────────────────────────────────────┘
                │ OTLP gRPC
                ▼
┌─────────────────────────────────────────────────────────┐
│  OTel Collector  :4317 (internal)  :4319 (external)     │
│  ─ traces  → Tempo                                      │
│  ─ logs    → Loki  (OTLP path)                          │
└───────────────┬──────────────┬──────────────────────────┘
                │              │
                ▼              ▼
             Tempo           Loki
          (trace store)   (log store)
                │              │
                └──────┬───────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│  Grafana                                                │
│  datasources: Loki · Tempo · Prometheus · TimescaleDB   │
│  dashboards:  Entity Inspector · Pipeline Logs ·        │
│               Platform Health · Error Entities ·        │
│               Sherlock Cost                             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Grafana Alloy  (Docker log sidecar)                    │
│  ─ scrapes container stdout                             │
│  ─ extracts helix fields → Loki  (sidecar path)         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Prometheus                                             │
│  scrapes: herald · sherlock · otel-collector ·         │
│           loki · tempo · node-exporter                  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  TimescaleDB (PostgreSQL)                               │
│  tables: entities · entity_events · entity_operations   │
│          instrument_memory · sherlock_usage             │
│          notification_issues · notification_silences    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Sherlock  (AI troubleshooting)                         │
│  ─ POST /diagnose → streams root-cause analysis         │
│  ─ queries herald API, Loki, Prometheus, GitHub        │
└─────────────────────────────────────────────────────────┘
```

## Ports

### Externally accessible

| Service | Port | Protocol | Notes |
|---|---|---|---|
| Herald (OTLP) | `4317` | gRPC plaintext | Instrument pipelines connect here |
| OTel Collector (logs) | `4319` | gRPC plaintext | For pipelines using `otlp=True` log shipping |
| Grafana | `3001` | HTTPS (Caddy) | Dashboard UI |
| UI | `443` | HTTPS (Caddy) | Entity Inspector web app |

### Internal / operator access only

| Service | Port | Notes |
|---|---|---|
| Herald HTTP API | `8080` | `GET /api/v1/entity/{id}/graph` |
| Herald metrics | `2112` | Prometheus scrape target |
| Sherlock metrics | `9102` | Prometheus scrape target |
| Prometheus | `9091` | Query UI |
| TimescaleDB | `5432` | Direct DB access |
| Loki | `3101` | Log ingestion and query |
| Tempo | `3201` | Trace query |

## Data flows

### Traces

```
Instrument  →  Herald :4317  →  OTel Collector :4317  →  Tempo
```

The herald enriches each span with resolved parent links before forwarding. Spans without `helix.entity.id` are forwarded unchanged.

### Logs — OTLP path

```
Instrument (otlp=True)  →  OTel Collector :4319  →  Loki
```

Used when no log sidecar is present. The instrument ships log records directly as OTLP.

### Logs — sidecar path

```
Instrument (otlp=False)  →  stdout JSON  →  Alloy  →  Loki
```

Used when Grafana Alloy (or another collector) is already running alongside containers. Alloy tails stdout and extracts helix fields. See [Log Collection](log-collection.md) for the required Alloy pipeline configuration.

### Metrics

```
Prometheus  ←  scrapes  ←  herald · sherlock · otel-collector · loki · tempo · node-exporter
```

Metrics are not shipped via OTLP — all services expose a Prometheus `/metrics` endpoint.

### Entity data

The herald writes directly to TimescaleDB for:

| Table | Written when |
|---|---|
| `entities` | A span with `helix.entity.id` arrives (idempotent upsert) |
| `entity_operations` | `helix.entity.is_operation = "true"` |
| `entity_events` | Any `helix.*` span event |

Grafana queries TimescaleDB directly via the PostgreSQL datasource for entity graph and event data.
