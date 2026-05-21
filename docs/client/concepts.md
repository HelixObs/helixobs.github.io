# Core Concepts

## Entities

An **entity** is any data product your pipeline creates or transforms — a raw data block, a candidate detection, a calibration solution, a science file. Each entity has:

- A **stable string ID** you choose. Use whatever natural key exists in your domain: a database primary key, a file path hash, a UUID. The only requirement is that it is unique within your instrument.
- A **stage** name — a short label for the processing step that created it (e.g. `"ingest"`, `"filter"`, `"aggregate"`).
- Optional **metadata** — a free-form dict stored in TimescaleDB alongside the entity.
- An optional list of **parent IDs** — the entities this one was derived from.

## Provenance

Parent IDs form a DAG. Declare them when creating an entity:

```python
# block-001 was produced by no parents (it is a root entity)
with tel.create("ingest", id="block-001"):
    run_ingest()

# candidate-42 was derived from block-001
with tel.create("search", id="candidate-42", parents=["block-001"]):
    run_search()

# event-7 was derived from multiple candidates
with tel.create("cluster", id="event-7", parents=["candidate-42", "candidate-43"]):
    run_cluster()
```

The herald resolves parent IDs to OTel span links server-side and stores the graph in TimescaleDB. The Grafana Entity Inspector renders it as a clickable DAG.

### Cross-process parents

Parents do not need to live in the same process or host. If the parent entity was created in a different process, the herald resolves the link server-side using a shared TraceStore. Simply pass the parent ID — no coordination required.

## Tokens

A **token** represents the lifecycle of one entity through one processing stage. It maps directly to an [OpenTelemetry](https://opentelemetry.io/) span. The span is started when you call `.start()` (or enter a `with` block) and ended when you call `.complete()` or `.error()` — or automatically on clean exit from a `with` block.

```python
token = tel.create("search", id="candidate-42", parents=["block-001"])
token.start()
# ... do work ...
token.complete(metadata={"score": 12.4, "frequency": 332.1})
```

### Attaching metadata

Metadata is not limited to a single call. You can attach values at any point in an entity's lifecycle — at creation, and again through subsequent operations as more information becomes available. Use `token.set_attribute()` inside the `with` block; the context manager calls `complete()` automatically on exit:

```python
# Entity comes into existence — metadata known upfront
with tel.create("ingest", id="block-001") as token:
    data = ingest()
    token.set_attribute("n_samples", len(data))
    token.set_attribute("beam", 42)

# Later stage — metadata reflects processing outcome
with tel.operate("search", entity_id="block-001") as token:
    result = search(data)
    token.set_attribute("score", result.score)
    token.set_attribute("frequency_mhz", result.freq)
    token.set_attribute("dm", result.dm)
```

When using the Explicit API you can also pass all metadata at once to `complete()`:

```python
token = tel.operate("search", entity_id="block-001")
token.start()
result = search(data)
token.complete(metadata={"score": result.score, "frequency_mhz": result.freq, "dm": result.dm})
```

All metadata is stored in TimescaleDB and is available on the **[Monitor page](../monitor)** — a configurable time-series view of entity metadata fields across all entities. This makes it straightforward to track pipeline health metrics (detection scores, SNR, processing latency) without setting up separate dashboards.

## Operations

An **operation** is work done on an entity that already exists — archiving, registration, replication, reprocessing. It differs from entity creation in two ways:

1. It does **not** create a new entity row — it records an `entity_operations` row instead.
2. The entity's TraceStore entry is not overwritten, so future provenance links to the entity are unaffected.

```python
# The entity "event-7" already exists. This records post-processing work on it.
with tel.operate("archive", entity_id="event-7") as t:
    write_to_archive("event-7")
    t.complete(metadata={"archive_path": "/data/event-7.h5"})
```

Use `operate()` whenever your pipeline does work on an entity that was created upstream — even in a different process or pipeline run.

## Events

Named domain events can be attached to any entity or operation:

```python
token.add_event("helix.event.classified", metadata={"label": "FRB", "dm": "348.8", "confidence": "0.97"})
```

Any event whose name starts with `helix.event.` is extracted by the herald and stored in the `entity_events` table, and appears on the Entity Inspector timeline in Grafana. Use this for scientifically notable signals — classification changes, quality flags, derived measurements — that you want queryable independently of the full trace.

### Triggering notifications from events

Events can also trigger **Slack messages and GitHub issues** if the event name is configured in your instrument's notification config. For example, to notify on every `helix.event.classified` event, your operator adds to the instrument YAML:

```yaml
notifications:
  events:
    helix.event.classified:
      slack:
        channel: "#detections"
        sample_window_seconds: 60
        max_per_window: 5
```

Contact your operator to configure notifications for specific event types.

## Errors

There are two error methods depending on whether the failure is terminal:

**Hard error** — `token.error(metadata)` — records `helix.error` and **ends the span**. Use when the operation cannot continue.

```python
token.error({"reason": "NFS timeout", "path": "/data/output.h5"})
```

**Soft error** — `token.add_error(metadata)` — records `helix.error` but **leaves the span open**. Use when a sub-step fails but the operation continues. Call `complete()` or `error()` when done.

```python
with tel.operate("post-process", entity_id=product_id) as token:
    for step in steps:
        try:
            step.run()
        except Exception as e:
            token.add_error({"step": step.name, "message": str(e)})
    # context manager calls complete() on clean exit
```

Both methods emit a `helix.error` event that the herald stores in `entity_events`.

### Notifications for errors

If your instrument is configured for error notifications, every `helix.error` event automatically triggers a **Slack message** and/or opens a **GitHub issue**. The herald deduplicates by error fingerprint — repeated identical errors update the existing issue body rather than creating noise. Rate limiting, silence rules, and auto-close behaviour are all configured by your operator per instrument.

A Slack alert includes the error message, entity ID, a direct link to the Entity Inspector, and a "Manage Silences" button. A GitHub issue tracks occurrence count, first/last seen, and the list of affected entities — updated on every recurrence.

No code changes are needed on your side. As long as `token.error()` or `token.add_error()` is called with a descriptive `metadata` dict, the notification system has everything it needs:

```python
token.error({
    "message": "NFS write failed",
    "path": "/data/output.h5",
    "stage": "archive",
})
```

## Herald

The **herald** is the HelixObs server component your client library sends spans to. It is a gRPC service that listens on port `4317` — the standard OpenTelemetry port — so no custom protocol is required on the pipeline side.

When your code calls `create()` or `operate()`, the `helixobs` library exports an OTLP span to the herald in the background. The herald then:

- **Resolves provenance** — matches each `parent_id` to a real OTel span link, even if the parent was created in a different process or host
- **Writes to TimescaleDB** — stores entity rows, operation records, and `helix.*` events in a queryable time-series database
- **Dispatches notifications** — sends Slack messages and opens GitHub issues for `helix.error` events, with dedup and rate limiting
- **Forwards spans** — passes the enriched batch to the downstream OTel Collector, which delivers traces to Tempo and logs to Loki

From a developer perspective, the herald is invisible: you configure its address once (`OTEL_EXPORTER_OTLP_ENDPOINT`) and the client handles the rest. Wherever these docs say "the herald resolves..." or "the herald stores...", this is the service doing that work.

## The instrument ID

The `instrument_id` is a short uppercase string identifying the telescope or instrument family (e.g. `"MY_TELESCOPE"`). It appears on every span and log line, is used as a Prometheus label, and scopes notification configs and silence rules. Choose one per instrument and keep it stable — changing it breaks Grafana queries and notification routing.
