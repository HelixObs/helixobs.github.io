# Getting Started

HelixObs is a self-hosted platform — there is no managed cloud tier. Before any pipeline can ship telemetry, an operator must deploy the HelixObs stack (herald, database, Grafana, etc.) and provide a connection endpoint.

**Which path applies to you?**

=== "I am an operator — I need to deploy the stack"

    Start with [Stack Architecture](operator/architecture.md) to understand what you are deploying, then follow [Deployment](operator/deployment.md) to bring it up with Docker Compose.

    Once the stack is running, share the herald address (and auth credentials if enabled) with your instrument teams so they can connect their pipelines.

=== "I am a developer — my operator has given me a herald address"

    You need:

    - The herald gRPC address (e.g. `herald.example.org:4317`)
    - An `instrument_id` for your pipeline (agree this with your operator)
    - Auth credentials if required (your operator will provide these)

    Install the client library:

    ```bash
    pip install helixobs
    ```

    Then instrument your pipeline:

    ```python
    from helixobs.setup import setup
    import logging

    log = logging.getLogger(__name__)

    tel = setup(
        "my-pipeline",
        instrument_id="MY_INST",
        endpoint="herald.example.org:4317",
        process_name="MY_INST/ingest",
    )

    with tel.create("ingest", id="product-001") as token:
        log.info("processing product")
        result = process("product-001")
        token.complete(metadata={"size_mb": result.size_mb})
    ```

    The entity `product-001` is now visible in the Grafana Entity Inspector, its logs are in Loki, and its trace is in Tempo.

    Continue to [Core Concepts](client/concepts.md) to understand entities and provenance, or jump to [Tracking Entities](client/tracking.md) for the full API.
