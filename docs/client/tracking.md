# Tracking Entities

The client library offers three integration layers. All three emit identical OTLP spans — choose whichever fits your code style.

## Setup

All three layers require a configured `Instrument`. Use `setup()` at application startup:

```python
from helixobs.setup import setup

tel = setup(
    "my-pipeline",
    instrument_id="MY_INST",
    endpoint="gateway:4317",
    process_name="MY_INST/ingest",
)
```

---

## Layer 0 — Token API

Explicit start/complete/error. Use this when entity creation and completion happen in different functions or callbacks.

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

## Layer 1 — Context Manager

`create()` and `operate()` return a `Token` that is also a context manager. The span starts on entry and `complete()` is called on clean exit; `error()` is called automatically if an exception propagates.

```python
with tel.create("search", id="candidate-42", parents=["block-001"]) as token:
    result = run_search()
    token.complete(metadata={"score": result.score})
```

```python
with tel.operate("archive", entity_id="event-7") as token:
    write_archive()
    token.complete(metadata={"path": "/data/event-7.h5"})
```

!!! tip
    Log **inside** the `with` block — the span is active, so every log line automatically carries `helix_entity_id` and `otel_trace_id`.

---

## Layer 2 — Decorator

The same `Token` returned by `create()` / `operate()` is also callable as a decorator. Pass a callable for `id` and `parents` so the entity ID is derived from the function arguments at call time.

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

## Provenance patterns

### Linear chain

```python
with tel.create("ingest", id="block-001") as t:
    t.complete()

with tel.create("search", id="candidate-42", parents=["block-001"]) as t:
    t.complete()
```

### N-to-1 (fan-in)

```python
# Many partial results → one aggregated output
partial_ids = ["result-001", "result-002", "result-003"]
with tel.create("aggregate", id="event-7", parents=partial_ids) as t:
    t.complete()
```

### Cross-process

Parent IDs can come from any upstream process — no shared memory required. The gateway resolves the link from its server-side TraceStore.

```python
# In process A:
with tel.create("ingest", id="block-001") as t:
    t.complete()

# In process B (different host):
with tel.create("search", id="candidate-42", parents=["block-001"]) as t:
    t.complete()
```

---

## Adding domain events

```python
with tel.create("classify", id="event-7", parents=["candidate-42"]) as token:
    label = classify()
    token.add_event("classified", attributes={"label": label, "confidence": "0.97"})
    token.complete()
```

Events named `helix.event.*` are stored in `entity_events` and appear in the Entity Inspector timeline.

---

## Child spans

For internal sub-steps that should appear in Tempo but do not need their own entity row, use `child_span()`:

```python
with tel.create("process", id="block-001") as token:
    with tel.child_span("filter", attributes={"filter.type": "bandpass"}):
        apply_filter()
    with tel.child_span("transform"):
        transform()
    token.complete()
```

Child spans inherit the current entity's trace context and appear in the Tempo waterfall but do not create additional entity rows in TimescaleDB.
