# Testing And Conformance

Phase 1.5 provides a protocol-level test environment before the real Slay the
Spire 2 mod exists.

It includes:

- deterministic mock bridge server;
- JSON Schema validation;
- OpenAPI endpoint contract;
- golden fixtures;
- protocol trace replay;
- SDK, CLI, and MCP adapter contract tests.

## Mock Bridge

The mock bridge is a fixture-driven implementation of the core protocol:

```text
GET  /health
GET  /capabilities
GET  /state
GET  /actions?state_id=...
POST /act
```

It must enforce the same protocol rules as the real bridge:

- common envelope;
- `/actions` `state_id` guard;
- `/act` validates `state_id` before `action_id`;
- `stale_state` returns 409;
- `invalid_action` returns 422;
- `game_busy`, `not_actionable`, and `unsupported_state` are reproducible;
- request IDs are emitted;
- trace events are written.

## Mock Modes

Deterministic mode:

- predictable `request_id`;
- fixed or incrementing `generated_at`;
- deterministic state transitions;
- used in CI.

Realtime mode:

- real timestamps;
- realistic delay and timeout simulation;
- used for manual/debug tests.

## Golden Fixture Shape

Fixtures define scenario entry points, terminal conditions, states, legal
actions, transitions, and optional behavior overrides.

```json
{
  "scenario_id": "combat_basic_ironclad_jaw_worm",
  "schema_version": "1.0.0",
  "initial_state_id": "mock:combat:0001",
  "terminal_state_ids": ["mock:combat:0003"],
  "terminal_reason": "combat_complete",
  "states": [],
  "actions_by_state": {},
  "transitions": {},
  "behaviors": {}
}
```

Busy behavior example:

```json
{
  "behaviors": {
    "mock:combat:0001": {
      "actions_response": {
        "status": 423,
        "error_code": "game_busy",
        "retryable": true,
        "becomes_actionable_after_requests": 2
      }
    }
  }
}
```

## Schema Source Of Truth

JSON Schema is authoritative for payload shapes.

OpenAPI is authoritative for endpoints, methods, status codes, and references
to schemas.

Rule: OpenAPI must reference JSON Schema artifacts or be generated from the
same source.

CI checks:

- schemas validate fixtures;
- OpenAPI references existing schemas;
- OpenAPI examples validate against schemas;
- all OpenAPI error responses reference known error codes;
- no undocumented error codes.

## Contract Tests

Envelope:

- every response has `ok`, `meta`, `data`, and `error`;
- success has `error = null`;
- failure has `data = null`;
- `meta.state_id` is nullable where appropriate.

State:

- `state_id` present for `/state`;
- `data.state_id == meta.state_id`;
- state-type payload invariants hold;
- refs are emitted only inside current state payload.

Actions:

- `/actions?state_id=current` returns executable actions;
- `/actions?state_id=old` returns `409 stale_state`;
- `action.requires_state_id == requested state_id`;
- disabled actions are omitted in primary response.

Act:

- valid action advances to expected state;
- stale state is checked before `action_id`;
- unknown action on current state returns `422 invalid_action`;
- busy scenario returns `423 game_busy`;
- unsupported scenario returns `501 unsupported_state`;
- action timeout is not treated as safe to auto-retry.

Trace:

- events validate against schema;
- `request_id` and `decision_id` correlate where applicable;
- selected action existed for the referenced `state_id`.

## Adapter Tests

Python SDK:

- `get_state()`;
- `list_actions(state_id)`;
- `act(state_id, action_id)`;
- auth handling;
- typed errors;
- retry for safe read operations;
- no automatic retry of `POST /act` after `action_timeout`.

CLI:

- JSON output by default;
- `sts2 state`;
- `sts2 actions --state-id ...`;
- `sts2 act --state-id ... --action-id ...`.

MCP adapter:

- `sts2_get_state`;
- `sts2_list_actions`;
- `sts2_act`;
- REST error envelope preserved in tool result;
- adapter does not invent action params.

The MVP MCP adapter uses the Python SDK instead of reimplementing REST client
logic. Future adapters in other languages may use generated clients, but they
must preserve the same envelope and error semantics.

## CI

Required in CI:

- schema validation;
- fixture validation;
- mock bridge tests;
- Python SDK tests;
- CLI tests;
- MCP adapter tests;
- trace replay tests.

Optional/local:

- real StS2 mod build;
- real game integration.

## Completion Criteria

Phase 1.5 is complete when:

- schemas and OpenAPI exist;
- mock bridge serves all core endpoints;
- at least five golden scenarios pass;
- Python SDK passes against mock;
- CLI passes against mock;
- MCP adapter exposes the three core tools against mock;
- TraceReplay validates a full mock run transcript;
- `stale_state`, `invalid_action`, `game_busy`, and `unsupported_state` are
  covered by tests.
