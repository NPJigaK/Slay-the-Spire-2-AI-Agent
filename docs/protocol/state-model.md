# State Model

`/state` returns a normalized agent-facing snapshot, not a raw game object
dump.

Every state includes common fields:

```text
state_id
state_type
mode
actionable
run
player
summary
```

Domain payloads are nullable and keyed by `state_type`.

## Common Shape

```json
{
  "state_id": "run_8f3a:combat:00018422",
  "state_type": "combat",
  "mode": "singleplayer",
  "actionable": true,
  "run": {
    "act": 1,
    "floor": 3,
    "ascension": 0,
    "gold": 99,
    "seed": null
  },
  "player": {
    "character": "IRONCLAD",
    "hp": 62,
    "max_hp": 70,
    "block": 0,
    "energy": 3,
    "max_energy": 3,
    "relics": [],
    "potions": [],
    "deck_summary": {}
  },
  "combat": null,
  "card_reward": null,
  "reward": null,
  "map": null,
  "event": null,
  "rest": null,
  "shop": null,
  "card_select": null,
  "treasure": null,
  "unknown": null,
  "summary": {
    "llm": "Short stable summary for prompt use."
  }
}
```

## State Types

MVP taxonomy:

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

## `actionable`

`actionable = true` means the bridge can list at least one executable action
for the current state.

`actionable = false` means the bridge can read state, but the primary contract
does not currently accept actions.

`/actions` behavior:

```text
state_id == current_state_id and actionable == true:
  200 OK with non-empty actions

state_id == current_state_id and actionable == false:
  409 not_actionable, 423 game_busy, or 501 unsupported_state

state_id != current_state_id:
  409 stale_state
```

The primary loop should not treat an empty successful action list as normal.

## Unknown vs Unsupported

`state_type = unknown` means the bridge cannot fully classify the current UI.
`/state` can still return diagnostics.

`unsupported_state` means the bridge can classify the state, but action coverage
is not implemented.

Example unknown payload:

```json
{
  "observed_screen": "SomeScreenNameIfKnown",
  "reason": "No matching state classifier",
  "actionable_reason": "unsupported"
}
```

## Payload Invariants

For `state_type = combat`, `combat` must be non-null.

For `state_type = unknown`, `unknown` must be non-null and domain payloads may
be null.

Domain-specific payloads should not be collapsed into a generic `screen` field
once a domain payload exists.

## State ID Semantics

`state_id` identifies the normalized observable snapshot.

Recommended format:

```text
<session_id>:<state_type>:<monotonic_revision>
```

Example:

```text
run_8f3a:combat:00018422
```

The monotonic revision increments when normalized observable state changes.

Examples:

- hand changes;
- energy changes;
- enemy HP, block, or intent changes;
- screen changes;
- reward options change;
- map options change;
- actionable phase changes.

It should not increment only because:

- an animation frame advanced;
- the cursor moved;
- a render-only UI effect changed.

## Refs

Refs are ephemeral and opaque:

- valid only within the `state_id` where they were emitted;
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
card_reward_1
map_node_3
event_option_0
shop_item_4
deck_card_12
```
