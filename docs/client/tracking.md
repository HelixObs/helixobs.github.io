# Tracking Entities

The client library offers three ways to track entities. All three emit identical OTLP spans — choose whichever fits your code structure.

## Setup

All three styles require a configured `Instrument`. Use `setup()` at application startup:

```python
from helixobs.setup import setup

tel = setup(
    "my-pipeline",
    instrument_id="MY_INST",
    endpoint="herald:4317",
    process_name="MY_INST/ingest",
)
```

---

## Context manager *(recommended)*

`create()` and `operate()` return a `Token` that works as a Python context manager. The span starts on entry and `complete()` is called automatically on clean exit; `error()` is called automatically if an exception propagates.

Use `token.set_attribute()` inside the block to attach metadata — no explicit `complete()` needed:

```python
with tel.create("search", id="candidate-42", parents=["block-001"]) as token:
    result = run_search()
    token.set_attribute("score", result.score)
```

```python
with tel.operate("archive", entity_id="event-7") as token:
    write_archive()
    token.set_attribute("path", "/data/event-7.h5")
```

!!! tip
    Log **inside** the `with` block — the span is active, so every log line automatically carries `helix_entity_id` and `otel_trace_id`.

---

## Decorator

The same `Token` is also usable as a decorator. Pass a callable for `id` and `parents` so the entity ID is derived from the function arguments at call time.

```python
@tel.create("search", id=lambda block_id, **_: block_id)
def search_block(block_id):
    result = run_search(block_id)
    return result

search_block("candidate-42")
```

Or pass a static ID if it is known at decoration time:

```python
@tel.operate("daily-report", entity_id="report-2026-05-20")
def generate_report():
    ...
```

---

## Explicit API

Use this when entity creation and completion happen in different functions or callbacks — for example, when a span is opened in one thread and closed in another.

```python
token = tel.create("search", id="candidate-42", parents=["block-001"])
token.start()

try:
    result = run_search()
    token.complete(metadata={"score": result.score})
except Exception as e:
    token.error({"message": str(e)})
```

For operations on existing entities:

```python
token = tel.operate("archive", entity_id="event-7")
token.start()
write_archive()
token.complete(metadata={"path": "/data/event-7.h5"})
```

---

## Provenance patterns

### Linear chain

```python
with tel.create("ingest", id="block-001"):
    run_ingest()

with tel.create("search", id="candidate-42", parents=["block-001"]):
    run_search()
```

### N-to-1 (fan-in)

```python
# Many partial results → one aggregated output
partial_ids = ["result-001", "result-002", "result-003"]
with tel.create("aggregate", id="event-7", parents=partial_ids):
    aggregate()
```

### Cross-process

Parent IDs can come from any upstream process — no shared memory required. The [herald](concepts.md#herald) resolves the link from its server-side TraceStore.

```python
# In process A:
with tel.create("ingest", id="block-001"):
    run_ingest()

# In process B (different host):
with tel.create("search", id="candidate-42", parents=["block-001"]):
    run_search()
```

---

## Adding domain events

```python
with tel.create("classify", id="event-7", parents=["candidate-42"]) as token:
    label = classify()
    token.add_event("classified", attributes={"label": label, "confidence": "0.97"})
```

Events named `helix.event.*` are stored in `entity_events` and appear in the [Entity Inspector](../operator/dashboards.md#entity-inspector) timeline.

---

## Child spans

For internal sub-steps that should appear in Tempo but do not need their own entity row, use `child_span()`:

```python
with tel.create("process", id="block-001"):
    with tel.child_span("filter", attributes={"filter.type": "bandpass"}):
        apply_filter()
    with tel.child_span("transform"):
        transform()
```

Child spans inherit the current entity's trace context and appear in the Tempo waterfall but do not create additional entity rows in TimescaleDB.

---

## Operations where the entity is discovered mid-execution

Some pipelines don't know which entity they're operating on until after fetching work from a queue or database. Pass no `entity_id` to `operate()` so the entire execution — including the discovery work — is captured in one trace, then call `token.set_entity_id()` once the entity is known.

```python
with tel.operate("stage-deletion") as op:
    # queue check and DB fetch are inside this trace
    if queue.is_full():
        log.info("queue full, skipping")
        return                          # trace closes without entity_id → passthrough

    dataset = await fetch_next_dataset()
    if dataset is None:
        return                          # same — no entity linked

    op.set_entity_id(dataset.name)      # all subsequent logs carry this entity's trace

    replicas = await fetch_replicas(dataset.id)
    await stage_work(replicas)
    op.add_event("helix.event.deletion-staged", {"num_replicas": str(len(replicas))})
```

The story in the Entity Inspector: "During trace `abc123` the pipeline operated on entity `dataset-x`." All logs — including those before `set_entity_id()` — are reachable from the entity's operation row via its `trace_id` link.

---

## Plain traces for infrastructure work

Use `tel.trace()` for work that has no entity — HTTP servers, periodic health checks, daemon loops — where you want log correlation by trace ID without any entity machinery. No extra imports needed.

```python
with tel.trace("process-request", attributes={"endpoint": "/api/status"}):
    with tel.child_span("auth-check"):
        verify_token(request)
    with tel.child_span("db-query"):
        result = db.query(...)
    log.info("request complete")    # otel_trace_id present → filterable in Loki
```

`child_span()` calls inside a `tel.trace()` block automatically inherit the trace context and share its `otelTraceID`. The herald forwards these spans unchanged — no entity rows are written.
