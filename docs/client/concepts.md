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
with tel.track("ingest", id="block-001") as t:
    t.complete()

# candidate-42 was derived from block-001
with tel.track("search", id="candidate-42", parents=["block-001"]) as t:
    t.complete()

# event-7 was derived from multiple candidates
with tel.track("cluster", id="event-7", parents=["candidate-42", "candidate-43"]) as t:
    t.complete()
```

The gateway resolves parent IDs to OTel span links server-side and stores the graph in TimescaleDB. The Grafana Entity Inspector renders it as a clickable DAG.

### Cross-process parents

Parents do not need to live in the same process or host. If the parent entity was created in a different process, the gateway resolves the link server-side using a shared TraceStore. Simply pass the parent ID — no coordination required.

## Tokens

A **token** represents the lifecycle of one entity through one processing stage. It maps directly to an [OpenTelemetry](https://opentelemetry.io/) span. The span is started when you call `.start()` (or enter a `with` block) and ended when you call `.complete()` or `.error()`.

```python
token = tel.create("search", id="candidate-42", parents=["block-001"])
token.start()
# ... do work ...
token.complete(metadata={"score": 12.4, "frequency": 332.1})
```

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
token.add_event("calibration-applied", metadata={"solution_id": "cal-2026-05-20"})
```

Any event whose name starts with `helix.event.` is extracted by the gateway and stored in the `entity_events` table. Use this for scientifically notable signals — classification changes, quality flags, derived measurements — that you want queryable independently of the full trace.

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

Both methods emit a `helix.error` event that the gateway stores in `entity_events` and uses to trigger notifications (Slack, GitHub Issues).

## The instrument ID

The `instrument_id` is a short uppercase string identifying the telescope or instrument family (e.g. `"MY_TELESCOPE"`). It appears on every span and log line, is used as a Prometheus label, and scopes notification configs and silence rules. Choose one per instrument and keep it stable — changing it breaks Grafana queries and notification routing.
