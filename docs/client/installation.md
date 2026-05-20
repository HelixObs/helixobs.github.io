# Installation

## Basic install

```bash
pip install helixobs
```

This includes everything needed for entity tracking and sidecar log delivery (`otlp=False`, the default).

## OTLP log shipping

If your pipeline runs without a sidecar log collector (Alloy or similar) and you want to ship logs directly to the OTel Collector, install the OTLP exporter:

```bash
pip install "helixobs[otlp]"
```

This adds `opentelemetry-exporter-otlp-proto-grpc`, required for `configure_logging(otlp=True)`.

## Development install

```bash
git clone https://github.com/HelixObs/client-python
cd client-python
pip install -e ".[dev]"
pytest
```

## Python version support

Python 3.9 and later.

## Which log mode should I use?

| Situation | Recommended mode |
|---|---|
| Running in a Docker environment where Grafana Alloy is already scraping container logs | `otlp=False` (default) — Alloy handles delivery |
| Greenfield service with no log sidecar | `otlp=True` — logs go directly to the OTel Collector |
| Running both Alloy and OTLP | Use `otlp=False` to avoid duplicate log delivery |

See [Structured Logging](logging.md) for details on both modes.
