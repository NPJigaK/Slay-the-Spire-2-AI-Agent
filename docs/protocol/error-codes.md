# Error Codes

All errors use the common response envelope:

```json
{
  "ok": false,
  "meta": {
    "schema_version": "1.0.0",
    "bridge_version": "0.1.0",
    "game_version": "0.103.2",
    "request_id": "req_123",
    "generated_at": "2026-04-27T00:00:00Z",
    "mode": "singleplayer",
    "state_id": "run_8f3a:combat:00018422"
  },
  "data": null,
  "error": {
    "code": "stale_state",
    "message": "Action requires an older state.",
    "details": {},
    "retryable": true
  }
}
```

HTTP status and `error.code` must agree. `error.details` is object-shaped and
code-specific. `retryable` is per occurrence.

## Error Table

| HTTP | Code | Meaning |
| ---: | --- | --- |
| 400 | `invalid_request` | Malformed JSON, missing field, or schema failure. |
| 401 | `unauthorized` | Missing, invalid, or malformed auth token. |
| 403 | `forbidden` | Authenticated request is denied by policy/config. |
| 404 | `unknown_endpoint` | Endpoint does not exist. |
| 409 | `stale_state` | Request refers to a non-current `state_id`. |
| 409 | `not_actionable` | Stable state exists, but no primary action is accepted. |
| 422 | `invalid_action` | `action_id` is not legal for the current state. |
| 423 | `game_busy` | Temporary transition, animation, loading, or enemy turn. |
| 429 | `rate_limited` | Action execution rate limit exceeded. |
| 500 | `internal_error` | Bridge bug or unexpected exception. |
| 501 | `unsupported_state` | Known state exists, but action coverage is not implemented. |
| 503 | `bridge_unavailable` | Game or bridge is not ready. |
| 504 | `action_timeout` | Queued action did not complete before timeout. |

## Validation Order For `/act`

```text
1. Authenticate.
2. Validate request schema.
3. Check policy and safety state.
4. Validate state_id.
5. Validate action_id against current legal actions.
6. Validate game is executable.
7. Enqueue on main thread.
8. Wait for completion or timeout.
9. Emit trace.
```

If `state_id` differs from the current state, return `409 stale_state` without
checking whether the submitted `action_id` is valid.

## `game_busy` vs `not_actionable`

`game_busy` is temporary. Retry after a delay may succeed.

Examples:

- transition;
- animation;
- loading;
- enemy turn resolving.

`not_actionable` is stable but not controllable through the primary contract.

Examples:

- bridge paused;
- manual takeover;
- no run active;
- supported state with no current executable actions.

## `action_timeout`

`action_timeout` must not be automatically retried by SDKs. The action may have
executed after the timeout response. Clients must refresh `/state` before
choosing the next action.

Example details:

```json
{
  "previous_state_id": "run_8f3a:combat:00018422",
  "current_state_id": "run_8f3a:combat:00018422",
  "may_have_executed": false,
  "required_next_step": "refresh_state"
}
```

If the outcome is unknown:

```json
{
  "may_have_executed": true,
  "outcome": "unknown",
  "required_next_step": "refresh_state"
}
```
