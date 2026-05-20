# Structured Logging

HelixObs injects entity context into every log line emitted while a span is active — entity ID, instrument ID, trace ID, and span ID. These fields make it possible to find all logs associated with a specific entity in Loki with a single query.

## Enabling logging

Call `setup()` with a `service_name` — it wires logging and tracing together:

```python
tel = setup("my-pipeline", instrument_id="MY_INST", endpoint="gateway:4317")
```

Or call `configure_logging()` directly if you need logs without traces:

```python
from helixobs.logging import configure_logging
configure_logging()  # sidecar mode (default)
```

## Two delivery modes

=== "Sidecar mode (default)"

    ```python
    tel = setup("my-pipeline", instrument_id="MY_INST", endpoint="gateway:4317")
    # otlp=False is the default — no extra argument needed
    ```

    Every log line is written as a single-line JSON object to **stdout**:

    ```json
    {"ts": "2026-05-20T11:18:00", "level": "info", "logger": "my.pipeline",
     "msg": "block ingested", "helix_entity_id": "block-001",
     "helix_instrument_id": "MY_INST", "otel_trace_id": "a1b2c3...",
     "otel_span_id": "d4e5f6...", "helix_process_name": "MY_INST/l1-ingest"}
    ```

    A **log collector** (e.g. Grafana Alloy) running alongside your containers tails stdout and ships to Loki. The operator is responsible for deploying and configuring Alloy — see [Log Collection](../operator/log-collection.md).

    **Choose this when:** Alloy (or another sidecar) is already running in your environment.

=== "OTLP mode"

    ```python
    tel = setup(
        "my-pipeline",
        instrument_id="MY_INST",
        endpoint="gateway:4317",
        otlp=True,
        log_endpoint="http://otel-collector:4317",  # or set OTEL_EXPORTER_OTLP_ENDPOINT
    )
    ```

    Log records are shipped directly to the **OTel Collector** over gRPC using `BatchLogRecordProcessor`. No sidecar required.

    **Choose this when:** your environment has no log sidecar, or you want a fully OTel-native pipeline.

    Requires `pip install "helixobs[otlp]"`.

## Which mode to use

| Situation | Mode |
|---|---|
| Alloy (or another log collector) is already scraping container stdout in your environment | `otlp=False` — Alloy handles delivery; using both causes duplicate logs |
| No sidecar present | `otlp=True` |
| Unsure | Ask your operator |

!!! warning "Avoid duplicate logs"
    If Alloy is scraping container stdout **and** you use `otlp=True`, logs arrive in Loki twice — once from each path. Use `otlp=False` whenever a sidecar is present.

## Fields injected into every log line

| Field | Source | Notes |
|---|---|---|
| `ts` | Log record | ISO 8601 timestamp |
| `level` | Log record | `info`, `warning`, `error`, `debug` |
| `logger` | Log record | Logger name (`__name__`) |
| `msg` | Log record | Log message |
| `src` | File + line | Repo-relative path, or GitHub permalink if `GITHUB_REPO` + `GIT_COMMIT_SHA` are set |
| `helix_entity_id` | Active OTel span | Empty if no active span |
| `helix_instrument_id` | Active OTel span | Empty if no active span |
| `helix_process_name` | `process_name` from `setup()` | Empty if not configured |
| `otel_trace_id` | Active OTel span | 32-hex trace ID |
| `otel_span_id` | Active OTel span | 16-hex span ID |

!!! note
    `helix_entity_id` is **not** a Loki stream label. One stream per entity would exhaust Loki's stream limit. It is stored as Loki structured metadata (OTLP path) or extracted by Alloy into structured metadata (sidecar path). Both paths make it queryable with `| helix_entity_id = \`...\`` without `| json`.

## Process naming

`process_name` scopes the **Pipeline Process Logs** Grafana dashboard. Use a slash-delimited hierarchy rooted at your instrument ID:

```
{INSTRUMENT_ID}/{pipeline}/{stage}
```

| Pipeline stage | `process_name` |
|---|---|
| Detection stage | `MY_INST/detect` |
| Filtering stage | `MY_INST/filter` |
| Aggregation stage | `MY_INST/aggregate` |
| Archiver | `MY_INST/post/archive` |
| Registration | `MY_INST/post/register` |

This gives you regex subtree queries in Loki:

```logql
{helix_process_name=~"MY_INST/.*"}           # all stages
{helix_process_name=~"MY_INST/post/.*"}      # all post-processing stages
```

## Logging while a span is active

Log **inside** the span to capture entity context:

```python
# Correct — span is active, helix_entity_id is injected
with tel.track("ingest", id="block-001") as token:
    log.info("ingesting block")
    ingest()
    log.info("done")
    token.complete()

# Wrong — span has ended, no entity context
with tel.track("ingest", id="block-001") as token:
    ingest()
    token.complete()
log.info("done")  # helix_entity_id missing
```

## GitHub source links

Set these environment variables to get clickable source-file permalinks in every log line:

```bash
export GITHUB_REPO=https://github.com/my-org/my-pipeline
export GIT_COMMIT_SHA=abc123def456
```

The `src` field becomes a full GitHub permalink:
```
https://github.com/my-org/my-pipeline/blob/abc123def456/pipeline/ingest.py#L42
```
