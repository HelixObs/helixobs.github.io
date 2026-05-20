# Log Collection

HelixObs supports two log delivery paths. The path used by each instrument pipeline determines what the operator must deploy and configure. Both paths ultimately land logs in Loki where the Grafana dashboards can query them — but they require different setup.

## Overview

| | Sidecar path (Alloy) | OTLP path |
|---|---|---|
| **Instrument config** | `otlp=False` (default) | `otlp=True` |
| **Log source** | Container stdout (JSON) | gRPC to OTel Collector |
| **Operator requirement** | Alloy running with the correct pipeline | OTel Collector running (already part of the stack) |
| **Dashboard compatibility** | Requires the Alloy pipeline below | Works out of the box |
| **Risk of duplication** | None (only one path active) | Duplicate logs if Alloy is also present |

!!! warning "Avoid running both paths simultaneously"
    If Alloy is scraping container stdout **and** the instrument uses `otlp=True`, the same log lines arrive in Loki twice — once from each path. Coordinate with instrument teams: use `otlp=False` wherever Alloy is present, `otlp=True` where it is not.

---

## Sidecar path — Grafana Alloy

The instrument writes JSON log lines to stdout. Alloy tails container stdout, extracts helix fields, and pushes to Loki.

### Why specific pipeline configuration is required

The HelixObs Grafana dashboards filter logs using structured metadata:

```logql
{helix_instrument_id=~".+"} | helix_entity_id = `<entity-id>`
{helix_process_name="MY_INST/ingest"} | level=~"error"
```

These queries rely on `helix_entity_id`, `trace_id`, and `helix_process_name` being promoted out of the JSON body. Without the pipeline below, those fields stay in the raw JSON text and the queries return no results.

### Required Alloy pipeline

Add the following to your Alloy configuration wherever helixobs-instrumented containers are running:

```alloy
loki.source.docker "containers" {
  host             = "unix:///var/run/docker.sock"
  targets          = discovery.docker.containers.targets
  forward_to       = [loki.process.helix_json.receiver]
  refresh_interval = "5s"
}

loki.process "helix_json" {
  // Extract helix fields from JSON log lines.
  // Non-JSON lines pass through unchanged.
  stage.json {
    expressions = {
      helix_instrument_id = "helix_instrument_id",
      helix_process_name  = "helix_process_name",
      level               = "level",
      helix_entity_id     = "helix_entity_id",
      otel_trace_id       = "otel_trace_id",
    }
  }

  // Low-cardinality stream labels — safe to index.
  // helix_entity_id must NOT be a stream label (one stream per entity
  // would exhaust Loki's stream limit).
  stage.labels {
    values = {
      helix_instrument_id = "",
      helix_process_name  = "",
      level               = "",
    }
  }

  // Promote entity ID and trace ID as structured metadata so dashboard
  // queries work identically to the OTLP path.
  stage.structured_metadata {
    values = {
      helix_entity_id = "",
      trace_id        = "otel_trace_id",   // renamed to match OTel convention
    }
  }

  forward_to = [loki.write.default.receiver]
}

loki.write "default" {
  endpoint {
    url = "http://<your-loki-host>:3100/loki/api/v1/push"
  }
}
```

Replace `<your-loki-host>` with the address of your Loki instance (e.g. `loki` if running in the same Docker Compose network, or your central Loki URL with `tenant_id` if forwarding to a shared instance).

### Forwarding to a central Loki

If your environment has a central Loki instance shared across clusters, add a second write target:

```alloy
loki.write "central" {
  endpoint {
    url       = "https://loki.example.org/loki/api/v1/push"
    tenant_id = "my-org"
  }
}

// In loki.process "helix_json":
forward_to = [loki.write.default.receiver, loki.write.central.receiver]
```

---

## OTLP path

The instrument ships log records directly to the OTel Collector over gRPC. No Alloy configuration is required — the OTel Collector is already part of the HelixObs stack and forwards to Loki via the OTLP endpoint.

The OTel Collector listens on:

- `:4317` internally (within the Docker Compose network)
- `:4319` on the host (externally accessible)

Instruments using `otlp=True` should set:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://<helixobs-host>:4319
```

or pass `log_endpoint` to `setup()`:

```python
tel = setup(
    "my-pipeline",
    instrument_id="MY_INST",
    endpoint="gateway.example.org:4317",
    otlp=True,
    log_endpoint="http://helixobs.example.org:4319",
)
```

The OTel Collector maps OTel log attributes to Loki structured metadata, so dashboard queries work without any extra configuration.

---

## Checking which path is active

Query Loki to verify logs are arriving and fields are present:

```logql
# Should return results if Alloy pipeline is working
{helix_instrument_id="MY_INST"} | helix_entity_id = `<a-known-entity-id>`

# Should return results for either path
{helix_instrument_id="MY_INST"} | level = `error`
```

If `{helix_instrument_id="MY_INST"}` returns nothing, either:

- Alloy is not running or not scraping those containers, or
- The `stage.labels` block is missing `helix_instrument_id`, or
- The instrument is using `otlp=True` but `OTEL_EXPORTER_OTLP_ENDPOINT` is wrong
