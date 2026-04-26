# Bridge Design

## Goals

Build a clean-room Slay the Spire 2 bridge that lets external AI agents:

- observe normalized singleplayer game state;
- list executable legal actions for that state;
- select one opaque `action_id`;
- execute that action through `/act`;
- receive consistent errors for stale, invalid, busy, and unsupported states;
- produce replayable protocol traces.

The MVP is combat-first. Non-combat screens may be represented and must fail
safely, but full non-combat progression is not required for the initial MVP.

## Non-goals

- Multiplayer control.
- Starting runs from the menu.
- In-process LLM calls from the mod.
- Remote network control.
- Wildcard CORS.
- Free-form command execution.
- Batching or macro actions.
- RL training.
- Full card database or strategy engine.
- Full event, shop, rest, and card selection coverage.
- Steam Workshop packaging.

## Approved Architecture

```text
Clean-room STS2 Bridge Mod
  -> Versioned REST API
  -> StateBuilder / LegalActionBuilder / ActionExecutor
  -> JSONL trace

Protocol Conformance Harness
  -> mock bridge server
  -> JSON Schema / OpenAPI / golden fixtures
  -> stale_state / invalid_action / timeout tests

Adapters
  -> Python SDK + agent harness
  -> CLI
  -> MCP adapter
  -> skill/prompt package
```

The Protocol Conformance Harness is part of the architecture, not just a test
utility. It guarantees compatibility for clients before and after the real mod
exists.

## Phase Roadmap

Phase 0: Existing bridge investigation spike

- Compare STS2MCP, STS2-Agent, and sts2-mcp with black-box behavior and source
  reading.
- Evaluate API surface, legal actions, state schema, action schema, logging,
  timeout behavior, and stale-state handling.
- Record adopted ideas, rejected ideas, unresolved risks, license notes, and
  do-not-copy notes.

Phase 1: Protocol and schema design

- Define `/health`, `/capabilities`, `/state`, `/actions`, and `/act`.
- Define `schema_version`, `bridge_version`, `game_version`, `state_id`,
  `action_id`, and `legal_actions`.
- Create JSON Schema, OpenAPI, golden fixtures, an error code table, and trace
  event schema.

Phase 1.5: Protocol conformance harness

- Build a deterministic mock bridge before the real mod.
- Test SDK, CLI, MCP adapter, trace replay, stale-state behavior, and error
  mapping against the mock.
- Keep CI useful without Slay the Spire 2 binaries.

Phase 2: Clean-room STS2 mod

- Implement a singleplayer bridge mod.
- Implement combat read-only state, combat legal actions, and combat `/act`.
- Keep StS2, Godot, BaseLib, and Harmony dependencies inside the bridge mod's
  game integration layer.
- Ensure external agents see only the versioned protocol.

Phase 3: Adapters

- Python SDK and agent harness.
- JSON-first CLI.
- MCP adapter exposing only core tools at first.
- Skill/prompt package as guidance, not protocol.
- TypeScript SDK only when needed.

## Protocol Contract

Core endpoints:

```text
GET  /health
GET  /capabilities
GET  /state
GET  /actions?state_id=<state_id>
POST /act
```

Primary loop:

```text
1. Client calls /state.
2. Client calls /actions?state_id=<state_id>.
3. Client selects one action_id.
4. Client calls /act with state_id + action_id.
5. Bridge rejects stale state with 409 stale_state.
```

All endpoints use a common envelope:

```json
{
  "ok": true,
  "meta": {
    "schema_version": "1.0.0",
    "bridge_version": "0.1.0",
    "game_version": "0.103.2",
    "request_id": "req_123",
    "generated_at": "2026-04-27T00:00:00Z",
    "mode": "singleplayer",
    "state_id": null
  },
  "data": {},
  "error": null
}
```

`meta.state_id` is optional. For `/state` and `/actions`, it must match
`data.state_id`. For successful `/act`, it should equal `data.new_state_id`
when known. For `/health` and `/capabilities`, it may be null.

`action_id` is opaque:

- unique within its `state_id`;
- executable only with its `requires_state_id`;
- may be human-readable;
- must not be parsed by clients;
- semantic meaning belongs in `kind`, `label`, `params`, and `tags`.

`/act` accepts only `state_id` and `action_id` in the primary contract.
`params` are descriptive metadata and are not resubmitted by clients.

## State Model

`/state` returns a normalized agent-facing snapshot, not a raw game object dump.
Every state includes common fields:

- `state_id`
- `state_type`
- `mode`
- `actionable`
- `run`
- `player`
- `summary`

Domain payloads are nullable and keyed by `state_type`:

```text
combat
card_reward
reward
map
event
rest
shop
card_select
treasure
unknown
```

MVP state types:

```text
menu
combat
card_reward
reward
map
event
rest
shop
card_select
treasure
unknown
```

`actionable = true` means the bridge can list at least one executable action
for the current state. The primary `/actions` response should not silently
return an empty successful action list for non-actionable states.

Refs are ephemeral and opaque:

- valid only within the emitting `state_id`;
- may be human-readable;
- must not be constructed by clients;
- may change after any successful `/act`.

Examples:

```text
player
enemy_0
hand_0
potion_1
reward_2
map_node_3
event_option_0
shop_item_4
```

## Action Model

`/actions` returns executable actions only in the primary contract. Disabled
actions are omitted. Debug views may later support
`GET /actions?include_disabled=true`.

Each action has:

- `action_id`
- `kind`
- `label`
- `requires_state_id`
- `params`
- `tags`
- optional `effects_hint`

`effects_hint` is best-effort, optional, incomplete, and non-authoritative. It
must not be used for rules validation.

MVP action taxonomy:

```text
combat.play_card
combat.use_potion
combat.end_turn

card_reward.pick
card_reward.skip

reward.claim
reward.proceed

map.choose_node

event.choose_option
event.advance_dialogue

rest.choose_option
rest.proceed

shop.buy
shop.remove_card
shop.proceed

card_select.pick
card_select.confirm
card_select.cancel

treasure.claim
treasure.proceed
```

Menu actions such as `menu.start_run` are outside the MVP.

## Error Model

All errors use the common envelope. HTTP status and `error.code` must agree.
`error.details` is object-shaped and code-specific. `retryable` is per
occurrence, not implied only by code.

Important validation order for `/act`:

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

If `state_id` is stale, return `409 stale_state` before validating `action_id`.
SDKs must not automatically retry `POST /act` after `action_timeout`; clients
must refresh `/state` before choosing the next action.

## Testing And Conformance

Phase 1.5 provides:

- deterministic mock bridge server;
- JSON Schema validation;
- OpenAPI endpoint contract;
- golden fixtures;
- protocol trace replay;
- SDK, CLI, and MCP adapter contract tests.

The mock bridge must enforce the same protocol rules as the real bridge:

- common envelope;
- `state_id` guard;
- `/act` validates `state_id` before `action_id`;
- `stale_state` returns 409;
- `invalid_action` returns 422;
- busy, not-actionable, and unsupported scenarios are reproducible;
- trace events are emitted.

Trace replay is protocol transcript replay, not game replay. It verifies that
selected actions existed in `/actions` for their `state_id`, that stale-state
handling is correct, and that `request_id` / `decision_id` correlation is valid.

## Security And Safety

The bridge is a local game-control API. Localhost is not treated as inherently
safe.

Release defaults:

- bind to `127.0.0.1` only;
- `auth_required = true`;
- Bearer token required for `/capabilities`, `/state`, `/actions`, and `/act`;
- `/health` is unauthenticated but minimal;
- wildcard CORS disabled;
- remote bind disabled;
- debug commands disabled;
- multiplayer actions disabled.

Security-specific errors:

```text
unauthorized -> 401
forbidden -> 403
rate_limited -> 429
```

Safety states:

```text
active
read_only
paused
manual_takeover
disabled
```

The bridge stores only a local control token. It never stores OpenAI, Claude,
or other provider keys. Authorization headers and API keys are redacted from
trace.

## Component Boundaries

The repository is a monorepo, but protocol artifacts are the dependency root.
No implementation defines an independent action or state schema.

Allowed dependencies:

```text
bridge-mod -> protocol
mock-bridge -> protocol
Python SDK -> protocol
CLI -> Python SDK
agent harness -> Python SDK
MCP adapter -> Python SDK or generated REST client
trace replay -> protocol
```

Forbidden dependencies:

```text
bridge-mod -> Python SDK / MCP / agent harness
adapters -> bridge-mod internals
agents -> bridge-mod internals
any component -> independent action schema
```

SDK boundary:

```text
Python SDK may handle:
  - auth header
  - request_id propagation
  - envelope parsing
  - typed errors
  - retry for safe read operations
  - optional retry for game_busy on /actions

Python SDK must not handle:
  - choosing actions
  - retrying POST /act automatically after action_timeout
  - interpreting action_id semantics
  - applying gameplay heuristics
```

Agent harness boundary:

```text
Agent Harness handles:
  - policy loop
  - model calls
  - rule-based fallback
  - manual policy
  - decision trace
  - benchmark metrics
```

MVP MCP adapter uses the Python SDK so auth, envelope parsing, typed errors,
and retry policy are not reimplemented. A future TypeScript MCP adapter may use
a generated TypeScript client, but it must preserve the same REST envelope and
error semantics.

Clean-room Bridge Mod internals should isolate volatile game dependencies:

```text
Game/
  Integration/
  StateBuilder/
  LegalActionBuilder/
  ActionExecutor/
  Refs/
  Classifiers/
```

## Repository Layout

Target layout:

```text
repo-root/
  README.md

  docs/
    adr/
    design/
    investigations/
    protocol/

  protocol/
    schemas/
    openapi/
    fixtures/

  generated/
    csharp/
    python/
    typescript/

  bridge-mod/
  mock-bridge/
  python/
  mcp-adapter/
  tools/
  scripts/
```

Generated code is generated from protocol artifacts, is not manually edited,
has a documented regeneration command, and should be checked by CI for staleness
once generation exists.

Source packages are not scaffolded in the design pass.

## MVP Completion Criteria

MVP definition:

```text
The MVP is a combat-first, protocol-stability-first milestone.
A local external agent can:
  - observe current singleplayer state;
  - list executable legal actions;
  - choose one opaque action_id;
  - execute through /act;
  - receive stale_state / invalid_action / busy errors consistently;
  - produce a replayable JSONL trace.
```

Phase 0 complete:

- existing bridge investigation and clean-room notes complete.

Phase 1 complete:

- protocol, schema, state, action, error, and trace design accepted.

Phase 1.5 complete:

- deterministic mock bridge, fixtures, SDK/CLI/MCP tests, TraceReplay, and CI
  without StS2 are in place.

Phase 2 complete:

- clean-room mod loads;
- `/health` and `/capabilities` work;
- combat state/actions/act work;
- main-thread execution is used;
- safe failure and trace validation work.

Phase 3 complete:

- Python agent harness;
- JSON-first CLI;
- small MCP adapter;
- skill/prompt guide.

## Quality Gates

Protocol:

- all examples validate against schemas;
- OpenAPI references existing schemas;
- every error code is documented;
- clients ignore unknown fields.

Safety:

- protected endpoints require Bearer token;
- `/health` unauthenticated output is minimal;
- read-only, paused, and manual takeover reject `/act`;
- `/act` timeout is not automatically retried;
- auth headers are absent from trace.

Behavior:

- executable actions only in primary `/actions`;
- no empty successful `/actions` for non-actionable current state;
- `/act` validates `state_id` before `action_id`;
- `action_id` and refs are treated as opaque;
- state revision does not advance for animation-only changes.

Mock/real parity:

- real bridge responses validate against the same schemas as mock bridge;
- real bridge uses the same error envelope and HTTP status mapping;
- real bridge trace validates with the same TraceReplay tooling.

Clean-room:

- Phase 0 notes cite sources;
- incompatible licensed code is not copied;
- clean-room implementation choices are documented.

## Open Questions

These remain open until implementation planning:

1. Implementation language/tooling for mock bridge.
2. Whether JSON Schema/OpenAPI are handwritten first or generated from typed
   models.
3. Which BaseLib / ModTemplate version Phase 2 should target.
4. Final default port number.
5. Auth token UX and exact per-platform config path.
6. State revision algorithm: hash, monotonic counter, or hybrid.
7. Whether to omit combat effect hints initially.
8. Minimal unknown diagnostics without raw dumps.
9. CI platform strategy.
10. MCP adapter runtime.
11. Trace storage path and rotation policy.
12. Real-game smoke test shape.
