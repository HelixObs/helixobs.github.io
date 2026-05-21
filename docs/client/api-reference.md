# API Reference

## `setup()` — recommended entry point

```python
from helixobs.setup import setup

tel = setup(
    service_name,
    *,
    instrument_id=None,
    endpoint="localhost:4317",
    insecure=True,
    otlp=False,
    log_endpoint=None,
    process_name=None,
    credential=None,
    auth_endpoint=None,
    instrument_class=Instrument,
)
```

Configures logging and returns a ready-to-use `Instrument` stamped with the same `service_name`.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `service_name` | `str` | — | OTel service name for traces and logs |
| `instrument_id` | `str` | — | Short instrument identifier (e.g. `"MY_INST"`). Required unless `instrument_class` owns its own ID |
| `endpoint` | `str` | `"localhost:4317"` | Herald gRPC address |
| `insecure` | `bool` | `True` | Disable TLS (set `False` in production with TLS) |
| `otlp` | `bool` | `False` | Ship logs via OTLP instead of stdout JSON |
| `log_endpoint` | `str\|None` | `None` | OTel Collector address for logs. Falls back to `OTEL_EXPORTER_OTLP_ENDPOINT`, then `http://localhost:4317` |
| `process_name` | `str\|None` | `None` | Pipeline process name for the Pipeline Logs dashboard. Use `INST_ID/pipeline/stage` convention |
| `credential` | `str\|Callable\|None` | `None` | Registration secret or callable returning one. Required when herald auth is enabled |
| `auth_endpoint` | `str\|None` | `None` | Herald auth token endpoint. Required when `credential` is set |
| `instrument_class` | `type` | `Instrument` | Subclass to instantiate instead of base `Instrument` |

---

## `Instrument`

### `Instrument.create(stage, *, id, parents=None) → Token`

Returns a `Token` for a new entity. Works as a context manager, decorator, or explicit API — see [Tracking Entities](tracking.md).

```python
# Context manager (recommended)
with tel.create("ingest", id="block-001", parents=["upstream"]) as token:
    token.set_attribute("size_mb", 42)
    # complete() called automatically on exit

# Decorator
@tel.create("ingest", id=lambda block_id, **_: block_id)
def ingest(block_id):
    ...

# Explicit API
token = tel.create("ingest", id="block-001", parents=["upstream"])
token.start()
token.complete()
```

### `Instrument.operate(operation, *, entity_id) → Token`

Returns a `Token` for work on an existing entity. Writes to `entity_operations`, not `entities`. Same three usage patterns as `create()`.

```python
with tel.operate("archive", entity_id="event-7") as token:
    write_archive()
    token.complete(metadata={"path": "/data/event-7.h5"})
```

### `Instrument.child_span(name, *, parent_id=None, attributes=None)`

Context manager for a child span that appears in the Tempo trace waterfall but does not create an entity row. Use for internal sub-steps within an entity's processing.

```python
with tel.create("process", id="block-001") as token:
    with tel.child_span("filter", attributes={"filter.type": "bandpass"}):
        apply_filter()
    token.complete()
```

### `Instrument.shutdown()`

Flushes pending spans and shuts down the exporter. Call at application exit if you need a clean flush — not required if the process exits normally.

---

## `Token`

### `token.start() → Token`

Starts the OTel span. Returns `self` for chaining. Called automatically when entering a `with` block.

### `token.complete(metadata=None)`

Ends the span in success state. `metadata` is a `dict` of JSON-serialisable values stored in TimescaleDB.

### `token.error(metadata=None)`

Records a `helix.error` span event, marks the span as failed, and **ends the span**. Use for hard failures — the operation cannot continue.

```python
token.error({"reason": "timeout", "stage": "archive"})
```

Triggers configured notifications (Slack, GitHub Issues).

### `token.add_error(metadata=None)`

Records a `helix.error` span event and marks the span as failed, but **leaves the span open**. Use for soft/recoverable failures where the operation continues and you will call `complete()` or `error()` later.

```python
with tel.operate("post-process", entity_id=product_id) as token:
    try:
        write_header()
    except Exception as e:
        token.add_error({"stage": "write-header", "message": str(e)})
    # context manager calls complete() on clean exit
```

### `token.add_event(name, attributes=None)`

Records a named span event. Events named `helix.event.*` are stored in `entity_events` and appear in the Entity Inspector timeline.

```python
token.add_event("helix.event.classified", attributes={"label": "candidate", "confidence": "0.97"})
```

### `token.set_attribute(key, value)`

Sets a span attribute. Values are coerced to strings.

```python
token.set_attribute("output.path", "/data/result.h5")
```

---

## `configure_logging()` — direct logging setup

```python
from helixobs.logging import configure_logging

configure_logging(otlp=False, service_name=None)
```

Prefer `setup()` for the common case. Use this directly only when you need logs without traces.

| Parameter | Default | Description |
|---|---|---|
| `otlp` | `False` | Ship logs via OTLP gRPC instead of stdout JSON |
| `service_name` | `None` | Required when `otlp=True` |

Safe to call multiple times — subsequent calls are no-ops.

---

## `install_context_fields()` — inject fields only

```python
from helixobs.logging import install_context_fields
install_context_fields()
```

Injects helix context fields into log records without adding or modifying any handlers. Use this when your pipeline already has its own logging setup and you only want `helix_entity_id`, `otel_trace_id`, etc. available in your existing format string.
