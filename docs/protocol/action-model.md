# Action Model

`/actions` returns executable actions only in the primary contract. Disabled
actions are omitted.

Agents do not construct commands. They choose one opaque `action_id` emitted by
the bridge and submit it to `/act` with the matching `state_id`.

## Action Shape

```json
{
  "action_id": "combat.play_card.hand_0.enemy_0",
  "kind": "combat.play_card",
  "label": "Play Strike on Jaw Worm",
  "requires_state_id": "run_8f3a:combat:00018422",
  "params": {
    "card_ref": "hand_0",
    "target_ref": "enemy_0"
  },
  "effects_hint": {
    "damage": 6,
    "block": 0
  },
  "tags": ["combat", "card", "damage", "targeted"]
}
```

## `action_id`

`action_id` is the executable token.

Rules:

- unique within the `state_id`;
- executable only with `requires_state_id`;
- may be human-readable;
- must be treated as opaque by clients;
- semantic meaning is in `kind`, `label`, `params`, and `tags`.

Clients must not parse `action_id` to construct behavior.

## Params

`params` are descriptive metadata for trace, debugging, and policy decisions.
Clients must not resubmit or modify `params` in the primary `/act` contract.

Primary `/act` request:

```json
{
  "state_id": "run_8f3a:combat:00018422",
  "action_id": "combat.play_card.hand_0.enemy_0",
  "client": {
    "name": "python-agent",
    "version": "0.1.0",
    "decision_id": "dec_000001"
  }
}
```

## Effects Hint

`effects_hint` is:

- optional;
- best-effort;
- possibly incomplete;
- non-authoritative;
- not guaranteed to match final game results;
- never used for rules validation.

Legal action correctness depends on whether the card, potion, or UI action is
currently executable, not on effect prediction.

## MVP Action Taxonomy

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

Future actions:

```text
menu.start_run
menu.continue_run
menu.abandon_run
debug.pause
debug.resume
```

## Disabled Actions

Disabled actions are not returned in primary `/actions`.

Future debug-only shape:

```text
GET /actions?include_disabled=true
```

Disabled action fields may include:

```json
{
  "enabled": false,
  "disabled_reason": "not_enough_energy"
}
```

This debug mode is outside the primary agent loop.
