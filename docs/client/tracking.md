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
    process_name="MY_INST/l1-search",  # optional; scopes Pipeline Logs dashboard
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
    token.complete(metadata={"score": result.score, "quality": result.quality})
except Exception as e:
    token.error(str(e))
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

The `with` block starts the span on entry and calls `.complete()` on clean exit, `.error()` on exception. This is the most common pattern.

```python
with tel.track("search", id="candidate-42", parents=["block-001"]) as token:
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

Wraps a function so each call tracks one entity. The decorated function receives a `token` keyword argument.

```python
@tel.stage("search")
def search_block(block_id, *, token):
    result = run_search(block_id)
    token.complete(metadata={"score": result.score})
    return result

# Call it like a normal function — entity ID is the first positional argument.
search_block("candidate-42", parents=["block-001"])
```

---

## Provenance patterns

### Linear chain

```python
with tel.track("ingest", id="block-001") as t:
    t.complete()

with tel.track("search", id="candidate-42", parents=["block-001"]) as t:
    t.complete()
```

### N-to-1 (fan-in)

```python
# Many partial results → one aggregated output
partial_ids = ["result-001", "result-002", "result-003"]
with tel.track("aggregate", id="event-7", parents=partial_ids) as t:
    t.complete()
```

### Cross-process

Parent IDs can come from any upstream process — no shared memory required. The gateway resolves the link from its server-side TraceStore.

```python
# In process A:
with tel.track("ingest", id="block-001") as t:
    t.complete()

# In process B (different host):
with tel.track("search", id="candidate-42", parents=["block-001"]) as t:
    t.complete()
```

---

## Adding domain events

```python
with tel.track("classify", id="event-7", parents=["candidate-42"]) as token:
    label = classify()
    token.add_event("classified", metadata={"label": label, "confidence": 0.97})
    token.complete()
```

Events named `helix.event.*` are stored in `entity_events` and appear in the Entity Inspector timeline.

---

## Subspan stages

Not every step needs to be a separate entity. Use child spans for internal sub-stages that are part of the same entity's lifecycle:

```python
with tel.track("process", id="block-001") as token:
    with tel._tracer.start_as_current_span("rfi-excision"):
        excise_rfi()
    with tel._tracer.start_as_current_span("transform"):
        transform()
    token.complete()
```

Child spans appear in the Tempo trace waterfall but do not create additional entity rows.
