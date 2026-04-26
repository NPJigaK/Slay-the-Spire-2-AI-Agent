# Trace Format

Trace is JSONL. It is protocol-adjacent contract and is used for debugging,
conformance, replay validation, and benchmark summaries.

Trace replay is protocol transcript replay, not game replay.

## Trace Producers

Bridge trace:

- `state`
- `actions`
- `act_request`
- `act_result`
- `error`
- optional `health`
- optional `capabilities`

Agent trace:

- `decision`
- `model_request`
- `model_response`
- `fallback`
- `retry`
- `manual_takeover`

Combined trace is bridge trace plus agent trace joined by `request_id`,
`decision_id`, and `state_id`.

MVP trace replay must work with bridge trace only and can enrich validation when
agent trace is present.

## Required Fields

Every event:

```text
type
ts
schema_version
bridge_version when produced by bridge
```

When applicable:

```text
request_id
state_id
decision_id
client
payload or action/result/error
```

## Example Events

State:

```json
{
  "type": "state",
  "ts": "2026-04-27T00:00:00Z",
  "schema_version": "1.0.0",
  "bridge_version": "0.1.0",
  "request_id": "req_001",
  "state_id": "run_8f3a:combat:00018422",
  "state_type": "combat",
  "payload": {}
}
```

Actions:

```json
{
  "type": "actions",
  "ts": "2026-04-27T00:00:01Z",
  "schema_version": "1.0.0",
  "bridge_version": "0.1.0",
  "request_id": "req_002",
  "state_id": "run_8f3a:combat:00018422",
  "count": 8,
  "payload": []
}
```

Decision:

```json
{
  "type": "decision",
  "ts": "2026-04-27T00:00:02Z",
  "schema_version": "1.0.0",
  "client": "python-agent",
  "decision_id": "dec_000001",
  "state_id": "run_8f3a:combat:00018422",
  "action": {
    "action_id": "combat.play_card.hand_0.enemy_0",
    "kind": "combat.play_card",
    "label": "Play Strike on Jaw Worm",
    "params": {
      "card_ref": "hand_0",
      "target_ref": "enemy_0"
    }
  }
}
```

Act result:

```json
{
  "type": "act_result",
  "ts": "2026-04-27T00:00:03Z",
  "schema_version": "1.0.0",
  "bridge_version": "0.1.0",
  "request_id": "req_003",
  "client": "python-agent",
  "decision_id": "dec_000001",
  "ok": true,
  "previous_state_id": "run_8f3a:combat:00018422",
  "new_state_id": "run_8f3a:combat:00018423",
  "action": {
    "action_id": "combat.play_card.hand_0.enemy_0",
    "kind": "combat.play_card",
    "label": "Play Strike on Jaw Worm",
    "params": {
      "card_ref": "hand_0",
      "target_ref": "enemy_0"
    }
  }
}
```

Error:

```json
{
  "type": "error",
  "ts": "2026-04-27T00:00:04Z",
  "schema_version": "1.0.0",
  "bridge_version": "0.1.0",
  "request_id": "req_004",
  "state_id": "run_8f3a:combat:00018422",
  "error": {
    "code": "stale_state",
    "message": "Action requires an older state.",
    "details": {},
    "retryable": true
  }
}
```

## Replay Validation

TraceReplay verifies:

- selected action existed in `/actions` for that `state_id`;
- `action_id` is interpreted only in context of `state_id`;
- stale-state handling is correct;
- invalid actions are counted;
- request/decision correlation is valid;
- terminal reason and summary metrics can be emitted.

Initial metrics:

```text
actions_taken
invalid_actions
stale_state_count
game_busy_count
unsupported_state_count
average_decision_ms
state_type_counts
terminal_reason
```

## Redaction

Bridge trace must redact:

- Authorization headers;
- local control token;
- API keys;
- provider credentials if accidentally present.

Agent trace is responsible for redacting:

- model provider credentials;
- prompt/model payloads when configured;
- raw model responses when configured.

Trace storage should use a per-user directory, not the game install directory by
default. File permissions should restrict access to the local user where
possible, and rotation must enforce `max_file_mb`.
