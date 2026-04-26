# Protocol Fixtures

This directory will contain golden fixtures for the mock bridge and conformance
tests.

Planned initial fixtures:

```text
combat-basic.json
card-reward-basic.json
stale-state.json
not-actionable.json
unsupported-state.json
busy-then-actionable.json
```

Fixture shape:

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

Fixtures are authoritative examples, not normative schema definitions. Payload
shape is defined by JSON Schema.
