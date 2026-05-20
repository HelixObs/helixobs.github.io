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
| `endpoint` | `str` | `"localhost:4317"` | Gateway gRPC address |
| `insecure` | `bool` | `True` | Disable TLS (set `False` in production with TLS) |
| `otlp` | `bool` | `False` | Ship logs via OTLP instead of stdout JSON |
| `log_endpoint` | `str\|None` | `None` | OTel Collector address for logs. Falls back to `OTEL_EXPORTER_OTLP_ENDPOINT`, then `http://localhost:4317` |
| `process_name` | `str\|None` | `None` | Pipeline process name for the Pipeline Logs dashboard. Use `INST/pipeline/stage` convention |
| `credential` | `str\|Callable\|None` | `None` | Registration secret or callable returning one. Required when gateway auth is enabled |
| `auth_endpoint` | `str\|None` | `None` | Gateway auth token endpoint. Required when `credential` is set |
| `instrument_class` | `type` | `Instrument` | Subclass to instantiate instead of base `Instrument` |

---

## `Instrument`

### `Instrument.create(stage, *, id, parents=None)`

Returns a `Token` for a new entity. The entity comes into existence when the token's span is started.

```python
token = tel.create("ingest", id="block-001", parents=["upstream-block"])
```

### `Instrument.operate(stage, *, entity_id)`

Returns a `Token` for work on an existing entity. Writes to `entity_operations`, not `entities`.

```python
token = tel.operate("archive", entity_id="event-7")
```

### `Instrument.track(stage, *, id, parents=None)`

Context manager. Calls `.start()` on entry, `.complete()` on clean exit, `.error(str(exc))` on exception.

```python
with tel.track("search", id="candidate-42", parents=["block-001"]) as token:
    token.complete(metadata={"score": 12.4})
```

### `Instrument.stage(stage_name)`

Decorator. The decorated function receives a `token` keyword argument. The first positional argument is used as the entity ID.

```python
@tel.stage("search")
def search(block_id, *, token):
    token.complete(metadata={"score": compute_score()})
```

---

## `Token`

### `token.start()`

Starts the OTel span. Registers the entity in the in-process TraceStore so children can resolve provenance links.

### `token.complete(metadata=None)`

Ends the span in success state. `metadata` is a `dict` of JSON-serialisable values stored in TimescaleDB.

### `token.error(message)`

Records a `helix.error` span event and ends the span in error state. Triggers configured notifications (Slack, GitHub Issues).

### `token.add_event(name, metadata=None)`

Records a named `helix.event.<name>` span event. Stored in `entity_events` and surfaced in the Entity Inspector timeline.

```python
token.add_event("classified", metadata={"label": "candidate", "confidence": 0.97})
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

Injects helix context fields into log records without adding or modifying any handlers. Use this when your pipeline already has its own logging setup (rotating file handlers, `dictConfig`, etc.) and you only want the `helix_entity_id`, `otel_trace_id`, etc. fields available in your existing format string.
