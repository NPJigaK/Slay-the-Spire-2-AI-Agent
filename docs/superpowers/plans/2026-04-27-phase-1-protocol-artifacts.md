# Phase 1 Protocol Artifacts Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first concrete protocol artifacts for the REST-first, legal-action-first STS2 bridge: JSON Schemas, OpenAPI, golden fixtures, and CI validation.

**Architecture:** `protocol/` remains the dependency root. JSON Schema is authoritative for payload shapes, OpenAPI is authoritative for endpoints/status codes/schema references, and fixtures are authoritative examples. No source packages for the bridge mod, SDK, CLI, MCP adapter, or mock bridge are scaffolded in this phase.

**Tech Stack:** JSON Schema Draft 2020-12, OpenAPI 3.1, Python validation tooling, GitHub Actions.

---

## File Structure

Create or modify only protocol artifacts, validation tooling, CI, and docs:

```text
.github/workflows/protocol.yml
requirements-dev.txt
tools/validate_protocol.py

protocol/schemas/envelope.schema.json
protocol/schemas/error.schema.json
protocol/schemas/action.schema.json
protocol/schemas/act-request.schema.json
protocol/schemas/state.schema.json
protocol/schemas/trace-event.schema.json
protocol/schemas/fixture.schema.json

protocol/openapi/sts2-bridge.openapi.yaml

protocol/fixtures/combat-basic.json
protocol/fixtures/stale-state.json
protocol/fixtures/invalid-action.json
protocol/fixtures/game-busy.json
protocol/fixtures/unsupported-state.json

README.md
protocol/schemas/README.md
protocol/fixtures/README.md
docs/protocol/testing-conformance.md
```

Do not create these directories in this phase:

```text
bridge-mod/
mock-bridge/
python/
mcp-adapter/
generated/
```

## Task 1: Add Validation Tooling

**Files:**
- Create: `requirements-dev.txt`
- Create: `tools/validate_protocol.py`
- Create: `.github/workflows/protocol.yml`

- [ ] **Step 1: Create the dev requirements file**

Create `requirements-dev.txt`:

```text
jsonschema==4.23.0
openapi-spec-validator==0.7.1
PyYAML==6.0.2
```

- [ ] **Step 2: Create the protocol validator**

Create `tools/validate_protocol.py`:

```python
from __future__ import annotations

import json
import re
import sys
from pathlib import Path

import yaml
from jsonschema import Draft202012Validator
from openapi_spec_validator import OpenAPIV31SpecValidator
from openapi_spec_validator.readers import read_from_filename
from referencing import Registry, Resource
from referencing.jsonschema import DRAFT202012


ROOT = Path(__file__).resolve().parents[1]
SCHEMA_DIR = ROOT / "protocol" / "schemas"
FIXTURE_DIR = ROOT / "protocol" / "fixtures"
OPENAPI_FILE = ROOT / "protocol" / "openapi" / "sts2-bridge.openapi.yaml"


def load_json(path: Path) -> object:
    with path.open("r", encoding="utf-8") as handle:
        return json.load(handle)


def load_yaml(path: Path) -> object:
    with path.open("r", encoding="utf-8") as handle:
        return yaml.safe_load(handle)


def validate_schema_file(path: Path) -> None:
    schema = load_json(path)
    Draft202012Validator.check_schema(schema)


def build_schema_registry() -> Registry:
    resources = []
    for schema_path in sorted(SCHEMA_DIR.glob("*.schema.json")):
        schema = load_json(schema_path)
        resources.append(
            (
                schema["$id"],
                Resource.from_contents(schema, default_specification=DRAFT202012),
            )
        )
    return Registry().with_resources(resources)


def validate_instance(schema_path: Path, instance_path: Path) -> None:
    schema = load_json(schema_path)
    instance = load_json(instance_path)
    validator = Draft202012Validator(schema, registry=build_schema_registry())
    errors = sorted(validator.iter_errors(instance), key=lambda item: item.json_path)
    if errors:
        formatted = "\n".join(f"{error.json_path}: {error.message}" for error in errors)
        raise AssertionError(f"{instance_path} failed {schema_path}:\n{formatted}")


def validate_openapi_refs(openapi: dict) -> None:
    schemas = {path.name for path in SCHEMA_DIR.glob("*.schema.json")}
    text = OPENAPI_FILE.read_text(encoding="utf-8")
    referenced = set(re.findall(r"\.\./schemas/([A-Za-z0-9_.-]+\.schema\.json)", text))
    missing = sorted(name for name in referenced if name not in schemas)
    if missing:
        raise AssertionError("OpenAPI references missing schema files: " + ", ".join(missing))
    known_error_codes = {
        "invalid_request",
        "unauthorized",
        "forbidden",
        "unknown_endpoint",
        "stale_state",
        "not_actionable",
        "invalid_action",
        "game_busy",
        "rate_limited",
        "internal_error",
        "unsupported_state",
        "bridge_unavailable",
        "action_timeout",
    }
    for path, methods in openapi.get("paths", {}).items():
        for method, operation in methods.items():
            for status, response in operation.get("responses", {}).items():
                code = response.get("x-error-code")
                if status.startswith(("4", "5")) and code not in known_error_codes:
                    raise AssertionError(
                        f"{method.upper()} {path} status {status} has unknown x-error-code {code!r}"
                    )


def main() -> int:
    for schema_path in sorted(SCHEMA_DIR.glob("*.schema.json")):
        validate_schema_file(schema_path)

    fixture_schema = SCHEMA_DIR / "fixture.schema.json"
    for fixture_path in sorted(FIXTURE_DIR.glob("*.json")):
        validate_instance(fixture_schema, fixture_path)

    openapi, base_uri = read_from_filename(str(OPENAPI_FILE))
    OpenAPIV31SpecValidator(openapi, base_uri=base_uri).validate()
    validate_openapi_refs(openapi)

    print("Protocol artifacts are valid.")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 3: Run the validator before artifacts exist**

Run:

```powershell
python -m pip install -r requirements-dev.txt
python tools/validate_protocol.py
```

Expected: FAIL because schema and OpenAPI files do not exist yet.

- [ ] **Step 4: Create the GitHub Actions workflow**

Create `.github/workflows/protocol.yml`:

```yaml
name: Protocol

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: python -m pip install -r requirements-dev.txt

      - name: Validate protocol artifacts
        run: python tools/validate_protocol.py
```

- [ ] **Step 5: Commit validation tooling**

Run:

```powershell
git add requirements-dev.txt tools/validate_protocol.py .github/workflows/protocol.yml
git commit -m "Add protocol validation tooling"
```

Expected: commit succeeds.

## Task 2: Add Envelope And Error Schemas

**Files:**
- Create: `protocol/schemas/envelope.schema.json`
- Create: `protocol/schemas/error.schema.json`

- [ ] **Step 1: Create the error schema**

Create `protocol/schemas/error.schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://npjigak.github.io/sts2-ai-agent/schemas/error.schema.json",
  "title": "STS2 Bridge Error",
  "type": "object",
  "required": ["code", "message", "details", "retryable"],
  "additionalProperties": false,
  "properties": {
    "code": {
      "type": "string",
      "enum": [
        "invalid_request",
        "unauthorized",
        "forbidden",
        "unknown_endpoint",
        "stale_state",
        "not_actionable",
        "invalid_action",
        "game_busy",
        "rate_limited",
        "internal_error",
        "unsupported_state",
        "bridge_unavailable",
        "action_timeout"
      ]
    },
    "message": { "type": "string", "minLength": 1 },
    "details": { "type": "object", "additionalProperties": true },
    "retryable": { "type": "boolean" }
  }
}
```

- [ ] **Step 2: Create the response envelope schema**

Create `protocol/schemas/envelope.schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://npjigak.github.io/sts2-ai-agent/schemas/envelope.schema.json",
  "title": "STS2 Bridge Response Envelope",
  "type": "object",
  "required": ["ok", "meta", "data", "error"],
  "additionalProperties": false,
  "properties": {
    "ok": { "type": "boolean" },
    "meta": {
      "type": "object",
      "required": [
        "schema_version",
        "bridge_version",
        "game_version",
        "request_id",
        "generated_at",
        "mode",
        "state_id"
      ],
      "additionalProperties": false,
      "properties": {
        "schema_version": { "type": "string", "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+$" },
        "bridge_version": { "type": "string", "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+$" },
        "game_version": { "type": ["string", "null"] },
        "request_id": { "type": "string", "minLength": 1 },
        "generated_at": { "type": "string", "format": "date-time" },
        "mode": {
          "type": "string",
          "enum": ["singleplayer", "multiplayer_unsupported", "menu", "unknown"]
        },
        "state_id": { "type": ["string", "null"] }
      }
    },
    "data": true,
    "error": {
      "anyOf": [
        { "type": "null" },
        { "$ref": "error.schema.json" }
      ]
    }
  },
  "allOf": [
    {
      "if": { "properties": { "ok": { "const": true } } },
      "then": { "properties": { "error": { "type": "null" } } }
    },
    {
      "if": { "properties": { "ok": { "const": false } } },
      "then": {
        "properties": {
          "data": { "type": "null" },
          "error": { "$ref": "error.schema.json" }
        }
      }
    }
  ]
}
```

- [ ] **Step 3: Validate the two schemas**

Run:

```powershell
@'
from pathlib import Path
from jsonschema import Draft202012Validator
import json
for path in Path("protocol/schemas").glob("*.schema.json"):
    Draft202012Validator.check_schema(json.loads(path.read_text(encoding="utf-8")))
    print(path)
'@ | python -
```

Expected: both schema paths print.

- [ ] **Step 4: Commit envelope and error schemas**

Run:

```powershell
git add protocol/schemas/envelope.schema.json protocol/schemas/error.schema.json
git commit -m "Add envelope and error schemas"
```

Expected: commit succeeds.

## Task 3: Add Action And Act Request Schemas

**Files:**
- Create: `protocol/schemas/action.schema.json`
- Create: `protocol/schemas/act-request.schema.json`

- [ ] **Step 1: Create the action schema**

Create `protocol/schemas/action.schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://npjigak.github.io/sts2-ai-agent/schemas/action.schema.json",
  "title": "STS2 Legal Action",
  "type": "object",
  "required": ["action_id", "kind", "label", "requires_state_id", "params", "tags"],
  "additionalProperties": false,
  "properties": {
    "action_id": { "type": "string", "minLength": 1 },
    "kind": {
      "type": "string",
      "enum": [
        "combat.play_card",
        "combat.use_potion",
        "combat.end_turn",
        "card_reward.pick",
        "card_reward.skip",
        "reward.claim",
        "reward.proceed",
        "map.choose_node",
        "event.choose_option",
        "event.advance_dialogue",
        "rest.choose_option",
        "rest.proceed",
        "shop.buy",
        "shop.remove_card",
        "shop.proceed",
        "card_select.pick",
        "card_select.confirm",
        "card_select.cancel",
        "treasure.claim",
        "treasure.proceed"
      ]
    },
    "label": { "type": "string", "minLength": 1 },
    "requires_state_id": { "type": "string", "minLength": 1 },
    "params": { "type": "object", "additionalProperties": true },
    "effects_hint": {
      "type": "object",
      "additionalProperties": true
    },
    "tags": {
      "type": "array",
      "items": { "type": "string", "minLength": 1 },
      "uniqueItems": true
    }
  }
}
```

- [ ] **Step 2: Create the act request schema**

Create `protocol/schemas/act-request.schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://npjigak.github.io/sts2-ai-agent/schemas/act-request.schema.json",
  "title": "STS2 Act Request",
  "type": "object",
  "required": ["state_id", "action_id"],
  "additionalProperties": false,
  "properties": {
    "state_id": { "type": "string", "minLength": 1 },
    "action_id": { "type": "string", "minLength": 1 },
    "client": {
      "type": "object",
      "required": ["name", "version"],
      "additionalProperties": false,
      "properties": {
        "name": { "type": "string", "minLength": 1 },
        "version": { "type": "string", "minLength": 1 },
        "decision_id": { "type": "string", "minLength": 1 }
      }
    }
  }
}
```

- [ ] **Step 3: Validate the schemas**

Run:

```powershell
@'
from pathlib import Path
from jsonschema import Draft202012Validator
import json
for path in [Path("protocol/schemas/action.schema.json"), Path("protocol/schemas/act-request.schema.json")]:
    Draft202012Validator.check_schema(json.loads(path.read_text(encoding="utf-8")))
    print(path)
'@ | python -
```

Expected: both schema paths print.

- [ ] **Step 4: Commit action schemas**

Run:

```powershell
git add protocol/schemas/action.schema.json protocol/schemas/act-request.schema.json
git commit -m "Add action request schemas"
```

Expected: commit succeeds.

## Task 4: Add State Schema

**Files:**
- Create: `protocol/schemas/state.schema.json`

- [ ] **Step 1: Create the state schema**

Create `protocol/schemas/state.schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://npjigak.github.io/sts2-ai-agent/schemas/state.schema.json",
  "title": "STS2 Normalized State",
  "type": "object",
  "required": ["state_id", "state_type", "mode", "actionable", "run", "player", "summary"],
  "additionalProperties": true,
  "properties": {
    "state_id": { "type": "string", "minLength": 1 },
    "state_type": {
      "type": "string",
      "enum": [
        "menu",
        "combat",
        "card_reward",
        "reward",
        "map",
        "event",
        "rest",
        "shop",
        "card_select",
        "treasure",
        "unknown"
      ]
    },
    "mode": {
      "type": "string",
      "enum": ["singleplayer", "multiplayer_unsupported", "menu", "unknown"]
    },
    "actionable": { "type": "boolean" },
    "run": {
      "type": "object",
      "additionalProperties": true,
      "properties": {
        "act": { "type": ["integer", "null"], "minimum": 0 },
        "floor": { "type": ["integer", "null"], "minimum": 0 },
        "ascension": { "type": ["integer", "null"], "minimum": 0 },
        "gold": { "type": ["integer", "null"], "minimum": 0 },
        "seed": { "type": ["string", "null"] }
      }
    },
    "player": {
      "type": "object",
      "additionalProperties": true,
      "properties": {
        "character": { "type": ["string", "null"] },
        "hp": { "type": ["integer", "null"], "minimum": 0 },
        "max_hp": { "type": ["integer", "null"], "minimum": 0 },
        "block": { "type": ["integer", "null"], "minimum": 0 },
        "energy": { "type": ["integer", "null"], "minimum": 0 },
        "max_energy": { "type": ["integer", "null"], "minimum": 0 },
        "relics": { "type": "array", "items": { "type": "object", "additionalProperties": true } },
        "potions": { "type": "array", "items": { "type": "object", "additionalProperties": true } },
        "deck_summary": { "type": "object", "additionalProperties": true }
      }
    },
    "combat": { "type": ["object", "null"], "additionalProperties": true },
    "card_reward": { "type": ["object", "null"], "additionalProperties": true },
    "reward": { "type": ["object", "null"], "additionalProperties": true },
    "map": { "type": ["object", "null"], "additionalProperties": true },
    "event": { "type": ["object", "null"], "additionalProperties": true },
    "rest": { "type": ["object", "null"], "additionalProperties": true },
    "shop": { "type": ["object", "null"], "additionalProperties": true },
    "card_select": { "type": ["object", "null"], "additionalProperties": true },
    "treasure": { "type": ["object", "null"], "additionalProperties": true },
    "unknown": { "type": ["object", "null"], "additionalProperties": true },
    "summary": {
      "type": "object",
      "required": ["llm"],
      "additionalProperties": true,
      "properties": {
        "llm": { "type": "string" }
      }
    }
  },
  "allOf": [
    {
      "if": { "properties": { "state_type": { "const": "combat" } } },
      "then": { "required": ["combat"], "properties": { "combat": { "type": "object" } } }
    },
    {
      "if": { "properties": { "state_type": { "const": "unknown" } } },
      "then": { "required": ["unknown"], "properties": { "unknown": { "type": "object" } } }
    }
  ]
}
```

- [ ] **Step 2: Validate the state schema**

Run:

```powershell
@'
from pathlib import Path
from jsonschema import Draft202012Validator
import json
path = Path("protocol/schemas/state.schema.json")
Draft202012Validator.check_schema(json.loads(path.read_text(encoding="utf-8")))
print(path)
'@ | python -
```

Expected: `protocol/schemas/state.schema.json` prints.

- [ ] **Step 3: Commit state schema**

Run:

```powershell
git add protocol/schemas/state.schema.json
git commit -m "Add normalized state schema"
```

Expected: commit succeeds.

## Task 5: Add Trace And Fixture Schemas

**Files:**
- Create: `protocol/schemas/trace-event.schema.json`
- Create: `protocol/schemas/fixture.schema.json`

- [ ] **Step 1: Create the trace event schema**

Create `protocol/schemas/trace-event.schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://npjigak.github.io/sts2-ai-agent/schemas/trace-event.schema.json",
  "title": "STS2 Trace Event",
  "type": "object",
  "required": ["type", "ts", "schema_version"],
  "additionalProperties": true,
  "properties": {
    "type": {
      "type": "string",
      "enum": [
        "state",
        "actions",
        "act_request",
        "act_result",
        "error",
        "health",
        "capabilities",
        "decision",
        "model_request",
        "model_response",
        "fallback",
        "retry",
        "manual_takeover"
      ]
    },
    "ts": { "type": "string", "format": "date-time" },
    "schema_version": { "type": "string", "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+$" },
    "bridge_version": { "type": "string" },
    "request_id": { "type": "string" },
    "state_id": { "type": "string" },
    "decision_id": { "type": "string" },
    "client": { "type": "string" },
    "payload": true,
    "action": { "$ref": "action.schema.json" },
    "error": { "$ref": "error.schema.json" }
  }
}
```

- [ ] **Step 2: Create the fixture schema**

Create `protocol/schemas/fixture.schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://npjigak.github.io/sts2-ai-agent/schemas/fixture.schema.json",
  "title": "STS2 Mock Bridge Fixture",
  "type": "object",
  "required": [
    "scenario_id",
    "schema_version",
    "initial_state_id",
    "terminal_state_ids",
    "terminal_reason",
    "states",
    "actions_by_state",
    "transitions"
  ],
  "additionalProperties": false,
  "properties": {
    "scenario_id": { "type": "string", "minLength": 1 },
    "schema_version": { "type": "string", "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+$" },
    "initial_state_id": { "type": "string", "minLength": 1 },
    "terminal_state_ids": {
      "type": "array",
      "items": { "type": "string", "minLength": 1 },
      "uniqueItems": true
    },
    "terminal_reason": { "type": "string", "minLength": 1 },
    "states": {
      "type": "array",
      "items": { "$ref": "state.schema.json" }
    },
    "actions_by_state": {
      "type": "object",
      "additionalProperties": {
        "type": "array",
        "items": { "$ref": "action.schema.json" }
      }
    },
    "transitions": {
      "type": "object",
      "additionalProperties": { "type": "string", "minLength": 1 }
    },
    "behaviors": {
      "type": "object",
      "additionalProperties": true
    }
  }
}
```

- [ ] **Step 3: Validate the schemas**

Run:

```powershell
@'
from pathlib import Path
from jsonschema import Draft202012Validator
import json
for path in [Path("protocol/schemas/trace-event.schema.json"), Path("protocol/schemas/fixture.schema.json")]:
    Draft202012Validator.check_schema(json.loads(path.read_text(encoding="utf-8")))
    print(path)
'@ | python -
```

Expected: both schema paths print.

- [ ] **Step 4: Commit trace and fixture schemas**

Run:

```powershell
git add protocol/schemas/trace-event.schema.json protocol/schemas/fixture.schema.json
git commit -m "Add trace and fixture schemas"
```

Expected: commit succeeds.

## Task 6: Add Golden Fixtures

**Files:**
- Create: `protocol/fixtures/combat-basic.json`
- Create: `protocol/fixtures/stale-state.json`
- Create: `protocol/fixtures/invalid-action.json`
- Create: `protocol/fixtures/game-busy.json`
- Create: `protocol/fixtures/unsupported-state.json`

- [ ] **Step 1: Create the combat basic fixture**

Create `protocol/fixtures/combat-basic.json` with two combat states, actions for the first state, and transitions to the second state:

```json
{
  "scenario_id": "combat_basic_ironclad_jaw_worm",
  "schema_version": "1.0.0",
  "initial_state_id": "mock:combat:0001",
  "terminal_state_ids": ["mock:combat:0002"],
  "terminal_reason": "one_action_applied",
  "states": [
    {
      "state_id": "mock:combat:0001",
      "state_type": "combat",
      "mode": "singleplayer",
      "actionable": true,
      "run": { "act": 1, "floor": 3, "ascension": 0, "gold": 99, "seed": null },
      "player": { "character": "IRONCLAD", "hp": 62, "max_hp": 70, "block": 0, "energy": 3, "max_energy": 3, "relics": [], "potions": [], "deck_summary": {} },
      "combat": { "turn": 1, "phase": "play", "enemies": [{ "ref": "enemy_0", "name": "Jaw Worm", "hp": 44, "max_hp": 44, "block": 0, "intent": { "type": "attack", "damage": 11, "hits": 1, "label": "11" }, "status": [] }], "hand": [{ "ref": "hand_0", "name": "Strike", "card_id": "STRIKE_R", "type": "attack", "cost": 1, "can_play": true, "targeting": "enemy", "description": "Deal 6 damage." }], "draw_pile_count": 8, "discard_pile_count": 0, "exhaust_pile_count": 0 },
      "card_reward": null,
      "reward": null,
      "map": null,
      "event": null,
      "rest": null,
      "shop": null,
      "card_select": null,
      "treasure": null,
      "unknown": null,
      "summary": { "llm": "Combat turn 1. Player has 3 energy. Strike can target Jaw Worm." }
    },
    {
      "state_id": "mock:combat:0002",
      "state_type": "combat",
      "mode": "singleplayer",
      "actionable": true,
      "run": { "act": 1, "floor": 3, "ascension": 0, "gold": 99, "seed": null },
      "player": { "character": "IRONCLAD", "hp": 62, "max_hp": 70, "block": 0, "energy": 2, "max_energy": 3, "relics": [], "potions": [], "deck_summary": {} },
      "combat": { "turn": 1, "phase": "play", "enemies": [{ "ref": "enemy_0", "name": "Jaw Worm", "hp": 38, "max_hp": 44, "block": 0, "intent": { "type": "attack", "damage": 11, "hits": 1, "label": "11" }, "status": [] }], "hand": [], "draw_pile_count": 8, "discard_pile_count": 1, "exhaust_pile_count": 0 },
      "card_reward": null,
      "reward": null,
      "map": null,
      "event": null,
      "rest": null,
      "shop": null,
      "card_select": null,
      "treasure": null,
      "unknown": null,
      "summary": { "llm": "Strike was played. Player has 2 energy." }
    }
  ],
  "actions_by_state": {
    "mock:combat:0001": [
      {
        "action_id": "combat.play_card.hand_0.enemy_0",
        "kind": "combat.play_card",
        "label": "Play Strike on Jaw Worm",
        "requires_state_id": "mock:combat:0001",
        "params": { "card_ref": "hand_0", "target_ref": "enemy_0" },
        "effects_hint": { "damage": 6 },
        "tags": ["combat", "card", "damage", "targeted"]
      },
      {
        "action_id": "combat.end_turn",
        "kind": "combat.end_turn",
        "label": "End turn",
        "requires_state_id": "mock:combat:0001",
        "params": {},
        "tags": ["combat", "turn"]
      }
    ],
    "mock:combat:0002": [
      {
        "action_id": "combat.end_turn",
        "kind": "combat.end_turn",
        "label": "End turn",
        "requires_state_id": "mock:combat:0002",
        "params": {},
        "tags": ["combat", "turn"]
      }
    ]
  },
  "transitions": {
    "mock:combat:0001|combat.play_card.hand_0.enemy_0": "mock:combat:0002"
  },
  "behaviors": {}
}
```

- [ ] **Step 2: Create the error scenario fixtures**

Create the four remaining files with this common shape, changing `scenario_id`, `terminal_reason`, and `behaviors` for each file:

```json
{
  "scenario_id": "stale_state",
  "schema_version": "1.0.0",
  "initial_state_id": "mock:combat:0002",
  "terminal_state_ids": ["mock:combat:0002"],
  "terminal_reason": "stale_state_rejected",
  "states": [
    {
      "state_id": "mock:combat:0002",
      "state_type": "combat",
      "mode": "singleplayer",
      "actionable": true,
      "run": { "act": 1, "floor": 3, "ascension": 0, "gold": 99, "seed": null },
      "player": { "character": "IRONCLAD", "hp": 62, "max_hp": 70, "block": 0, "energy": 2, "max_energy": 3, "relics": [], "potions": [], "deck_summary": {} },
      "combat": { "turn": 1, "phase": "play", "enemies": [], "hand": [], "draw_pile_count": 8, "discard_pile_count": 1, "exhaust_pile_count": 0 },
      "card_reward": null,
      "reward": null,
      "map": null,
      "event": null,
      "rest": null,
      "shop": null,
      "card_select": null,
      "treasure": null,
      "unknown": null,
      "summary": { "llm": "Current state is mock:combat:0002." }
    }
  ],
  "actions_by_state": {
    "mock:combat:0002": [
      {
        "action_id": "combat.end_turn",
        "kind": "combat.end_turn",
        "label": "End turn",
        "requires_state_id": "mock:combat:0002",
        "params": {},
        "tags": ["combat", "turn"]
      }
    ]
  },
  "transitions": {},
  "behaviors": {
    "act_request": {
      "state_id": "mock:combat:0001",
      "action_id": "combat.end_turn",
      "expected_status": 409,
      "expected_error_code": "stale_state"
    }
  }
}
```

For `invalid-action.json`, use:

```json
"scenario_id": "invalid_action",
"terminal_reason": "invalid_action_rejected",
"behaviors": {
  "act_request": {
    "state_id": "mock:combat:0002",
    "action_id": "combat.play_card.hand_9.enemy_0",
    "expected_status": 422,
    "expected_error_code": "invalid_action"
  }
}
```

For `game-busy.json`, use:

```json
"scenario_id": "game_busy",
"terminal_reason": "busy_then_actionable",
"behaviors": {
  "mock:combat:0002": {
    "actions_response": {
      "status": 423,
      "error_code": "game_busy",
      "retryable": true,
      "becomes_actionable_after_requests": 2
    }
  }
}
```

For `unsupported-state.json`, set `state_type` to `card_select`, `actionable` to `false`, `combat` to `null`, `card_select` to `{ "mode": "upgrade_select" }`, and use:

```json
"scenario_id": "unsupported_state",
"terminal_reason": "unsupported_state_rejected",
"behaviors": {
  "actions_response": {
    "status": 501,
    "error_code": "unsupported_state",
    "retryable": false
  }
}
```

- [ ] **Step 3: Run fixture validation**

Run:

```powershell
python tools/validate_protocol.py
```

Expected: FAIL because OpenAPI does not exist yet. Fixture/schema validation should not be the failing part.

- [ ] **Step 4: Commit fixtures**

Run:

```powershell
git add protocol/fixtures/*.json
git commit -m "Add initial protocol fixtures"
```

Expected: commit succeeds.

## Task 7: Add OpenAPI Endpoint Contract

**Files:**
- Create: `protocol/openapi/sts2-bridge.openapi.yaml`

- [ ] **Step 1: Create the OpenAPI contract**

Create `protocol/openapi/sts2-bridge.openapi.yaml`:

```yaml
openapi: 3.1.0
info:
  title: STS2 AI Bridge API
  version: 1.0.0
servers:
  - url: http://127.0.0.1:17862
security:
  - bearerAuth: []
paths:
  /health:
    get:
      summary: Minimal bridge health
      security: []
      responses:
        "200":
          description: Minimal unauthenticated health response.
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "500":
          description: Internal bridge error.
          x-error-code: internal_error
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
  /capabilities:
    get:
      summary: Authenticated bridge capabilities.
      responses:
        "200":
          description: Supported states, actions, safety, and feature flags.
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "401":
          description: Missing or invalid token.
          x-error-code: unauthorized
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
  /state:
    get:
      summary: Current normalized state snapshot.
      responses:
        "200":
          description: Current state.
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "401":
          description: Missing or invalid token.
          x-error-code: unauthorized
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "503":
          description: Bridge unavailable.
          x-error-code: bridge_unavailable
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
  /actions:
    get:
      summary: List executable legal actions for the current state.
      parameters:
        - name: state_id
          in: query
          required: false
          schema:
            type: string
      responses:
        "200":
          description: Legal actions for the current state.
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "401":
          description: Missing or invalid token.
          x-error-code: unauthorized
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "409":
          description: Stale or not-actionable state.
          x-error-code: stale_state
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "423":
          description: Game is temporarily busy.
          x-error-code: game_busy
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "501":
          description: Known but unsupported state.
          x-error-code: unsupported_state
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
  /act:
    post:
      summary: Execute one opaque action_id for the current state.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: ../schemas/act-request.schema.json
      responses:
        "200":
          description: Action accepted and executed.
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "400":
          description: Invalid request body.
          x-error-code: invalid_request
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "401":
          description: Missing or invalid token.
          x-error-code: unauthorized
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "403":
          description: Operation forbidden by policy.
          x-error-code: forbidden
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "409":
          description: Stale or not-actionable state.
          x-error-code: stale_state
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "422":
          description: Unknown action for the current state.
          x-error-code: invalid_action
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "423":
          description: Game is temporarily busy.
          x-error-code: game_busy
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "429":
          description: Action rate limit exceeded.
          x-error-code: rate_limited
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "501":
          description: Known but unsupported state.
          x-error-code: unsupported_state
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
        "504":
          description: Queued action did not complete before timeout.
          x-error-code: action_timeout
          content:
            application/json:
              schema:
                $ref: ../schemas/envelope.schema.json
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
```

- [ ] **Step 2: Run full protocol validation**

Run:

```powershell
python tools/validate_protocol.py
```

Expected: PASS with `Protocol artifacts are valid.`

- [ ] **Step 3: Commit OpenAPI**

Run:

```powershell
git add protocol/openapi/sts2-bridge.openapi.yaml
git commit -m "Add OpenAPI protocol contract"
```

Expected: commit succeeds.

## Task 8: Update Protocol Docs And README Links

**Files:**
- Modify: `README.md`
- Modify: `protocol/schemas/README.md`
- Modify: `protocol/fixtures/README.md`
- Modify: `docs/protocol/testing-conformance.md`

- [ ] **Step 1: Update `protocol/schemas/README.md`**

Replace the planned-file wording with this:

````markdown
# Protocol Schemas

This directory contains JSON Schema artifacts for the STS2 bridge protocol.

JSON Schema is authoritative for payload shapes.

Current files:

- `envelope.schema.json`
- `error.schema.json`
- `action.schema.json`
- `act-request.schema.json`
- `state.schema.json`
- `trace-event.schema.json`
- `fixture.schema.json`

Validation:

```powershell
python -m pip install -r requirements-dev.txt
python tools/validate_protocol.py
```
````

- [ ] **Step 2: Update `protocol/fixtures/README.md`**

Replace the planned-file wording with this:

````markdown
# Protocol Fixtures

This directory contains golden fixtures for the mock bridge and conformance
tests.

Fixtures are authoritative examples, not normative schema definitions. Payload
shape is defined by JSON Schema.

Current fixtures:

- `combat-basic.json`
- `stale-state.json`
- `invalid-action.json`
- `game-busy.json`
- `unsupported-state.json`

Validation:

```powershell
python -m pip install -r requirements-dev.txt
python tools/validate_protocol.py
```
````

- [ ] **Step 3: Add artifact links to README**

Add these bullets under the design docs list in `README.md`:

```markdown
- [OpenAPI contract](protocol/openapi/sts2-bridge.openapi.yaml)
- [Protocol schemas](protocol/schemas/)
- [Protocol fixtures](protocol/fixtures/)
```

- [ ] **Step 4: Run validation and markdown sanity checks**

Run:

```powershell
python tools/validate_protocol.py
git diff --check
```

Expected:

```text
Protocol artifacts are valid.
```

`git diff --check` exits with code 0.

- [ ] **Step 5: Commit docs updates**

Run:

```powershell
git add README.md protocol/schemas/README.md protocol/fixtures/README.md docs/protocol/testing-conformance.md
git commit -m "Document protocol artifact validation"
```

Expected: commit succeeds.

## Task 9: Create Phase 1 And Phase 1.5 GitHub Backlog

**Files:**
- No repository files changed.

This task creates the issue backlog recommended by the design review. The local desktop environment did not have the GitHub CLI installed when this plan was written, so use the GitHub connector, GitHub web UI, or install `gh` before running CLI commands.

- [ ] **Step 1: Create Phase 1 issues**

Create these GitHub issues in `NPJigaK/Slay-the-Spire-2-AI-Agent`:

```text
#1 Define protocol JSON Schemas
#2 Define OpenAPI endpoint contract
#3 Create golden fixture format
#4 Create combat-basic fixture
#5 Create stale-state fixture
#6 Create invalid-action fixture
#7 Create game-busy fixture
#8 Create unsupported-state fixture
```

Issue body template:

```markdown
## Scope

Implement the artifact described by the issue title for Phase 1 protocol artifacts.

## Acceptance Criteria

- Artifact exists in the path defined by `docs/design/bridge-design.md`.
- `python tools/validate_protocol.py` passes.
- `git diff --check` passes.
- No bridge mod, SDK, CLI, MCP adapter, or mock bridge source package is scaffolded.
```

- [ ] **Step 2: Create Phase 1.5 issues**

Create these GitHub issues in `NPJigaK/Slay-the-Spire-2-AI-Agent`:

```text
#9 Choose mock bridge runtime
#10 Implement deterministic mock bridge
#11 Implement TraceReplay validator
#12 Add CI for schema + fixtures + mock bridge
```

Issue body template:

```markdown
## Scope

Prepare or implement the Phase 1.5 conformance harness work described by the issue title.

## Acceptance Criteria

- Work follows `docs/protocol/testing-conformance.md`.
- Mock/real parity rules remain intact.
- CI remains useful without Slay the Spire 2 installed.
- No clean-room bridge mod source is scaffolded before the Phase 2 plan.
```

- [ ] **Step 3: Record issue numbers in the final execution note**

After creating issues, include the created issue numbers and URLs in the execution summary.

## Task 10: Final Verification

**Files:**
- Verify all modified files.

- [ ] **Step 1: Run full validation**

Run:

```powershell
python tools/validate_protocol.py
git diff --check
git status --short
```

Expected:

```text
Protocol artifacts are valid.
```

`git diff --check` exits with code 0. `git status --short` is clean after commits.

- [ ] **Step 2: Inspect commit history**

Run:

```powershell
git log --oneline -8
```

Expected: recent commits show validation tooling, schemas, fixtures, OpenAPI, and docs updates.

- [ ] **Step 3: Stop before source scaffolding**

Confirm these paths do not exist:

```powershell
Test-Path bridge-mod
Test-Path mock-bridge
Test-Path python
Test-Path mcp-adapter
```

Expected: each command prints `False`.
