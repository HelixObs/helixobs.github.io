# For Scientists & Developers

This section covers everything you need to instrument a scientific pipeline with HelixObs.

## What you need to know

HelixObs revolves around two ideas:

**Entities** are data products — anything your pipeline creates, transforms, or consumes. Each entity has a stable string ID chosen by you (a database key, a filename, a UUID). The ID follows the entity across every pipeline stage, compute node, and instrument.

**Provenance** is the graph of which entities produced which. You declare parent IDs when creating an entity; the gateway assembles these into a queryable DAG.

## Pages in this section

| | |
|---|---|
| [Core Concepts](concepts.md) | Entities, tokens, operations, the provenance DAG |
| [Installation](installation.md) | pip install, optional dependencies |
| [Tracking Entities](tracking.md) | Three integration layers: Token API, context manager, decorator |
| [Structured Logging](logging.md) | Two delivery modes — sidecar (Alloy) and OTLP — and what each requires |
| [Authentication](auth.md) | Connecting to a gateway that requires credentials |
| [API Reference](api-reference.md) | Full reference for `setup()`, `Instrument`, `Token` |

## Quick orientation

```
helixobs.setup()          ← start here; returns an Instrument
  │
  ├── Instrument.create()     ← new entity coming into existence
  ├── Instrument.operate()    ← work on an existing entity
  ├── Instrument.track()      ← context manager shorthand (create + operate)
  └── Instrument.stage()      ← decorator shorthand
          │
          └── Token
                ├── .start()
                ├── .complete(metadata={...})
                ├── .error(message)
                └── .add_event(name, metadata={...})
```
