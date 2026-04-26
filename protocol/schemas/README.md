# Protocol Schemas

This directory will contain JSON Schema artifacts for the STS2 bridge protocol.

Planned files:

```text
envelope.schema.json
state.schema.json
action.schema.json
act-request.schema.json
error.schema.json
trace-event.schema.json
fixture.schema.json
```

Rules:

- JSON Schema is authoritative for payload shapes.
- OpenAPI references these schemas or is generated from the same source.
- Generated models must not diverge from these artifacts.
- Fixtures and examples must validate against these schemas.

No schema files are added yet because this pass records the accepted design
without scaffolding implementation artifacts.
