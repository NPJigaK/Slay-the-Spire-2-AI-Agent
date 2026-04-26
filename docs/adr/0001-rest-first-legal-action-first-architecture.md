# ADR-0001: REST-first, legal-action-first STS2 bridge architecture

Status: Accepted

Date: 2026-04-27

## Context

The project will build a clean-room Slay the Spire 2 bridge for external AI
agents. Existing projects such as STS2MCP, STS2-Agent, and sts2-mcp show that
local bridge control is feasible, but the final implementation should own its
protocol and avoid structural dependency on any existing bridge implementation.

The system needs to support multiple agent types: prompt-based LLM agents,
rule-based policies, local models, benchmark runners, MCP clients, and shell or
CLI automation. The protocol must be stable enough for tests, replay, adapters,
and future mod maintenance while Slay the Spire 2 remains subject to Early
Access changes.

## Decision

- Implement a clean-room STS2 Bridge Mod.
- Use a versioned REST API as the primary interface.
- Treat MCP, CLI, and skill/prompt packages as adapter layers.
- Require agents to choose from `/actions` and submit only opaque `action_id`
  tokens to `/act`.
- Do not include free-form command execution in the primary contract.
- Reject stale actions through `state_id` validation in `/act`.
- Include a Protocol Conformance Harness as part of the core architecture.
- Make the MVP singleplayer and combat-first.

Approved architecture:

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

Approved phase structure:

```text
Phase 0: Existing bridge investigation spike
Phase 1: Protocol and schema design
Phase 1.5: Mock bridge + conformance tests
Phase 2: Clean-room STS2 mod
Phase 3: Python/CLI/MCP/skill adapters
```

Core contract:

```text
GET  /state
GET  /actions
POST /act

Invariant:
  - /actions returns executable legal actions for a specific state_id.
  - agents select action_id only.
  - /act validates state_id before action_id.
  - stale state returns 409 stale_state.
  - free-form command is not part of the primary contract.
```

## Consequences

Positive:

- The core protocol works for LLM, rule-based, RL, CLI, and MCP clients.
- The mod remains free of provider SDKs and API keys.
- Tests and adapters can be developed against the mock bridge before the real
  mod exists.
- Early Access breakage can be localized to game integration code while the
  external protocol remains stable.

Tradeoffs:

- The project must design and maintain protocol artifacts before feature work.
- MCP clients require an adapter rather than speaking directly to the mod.
- The first MVP is narrower than a full game-playing system.

## Clean-room Rule

Phase 0 may document observed behavior, API shape, feature coverage, license
notes, and protocol-level ideas from existing projects. Implementation must not
copy incompatible source code, line-by-line structure, class/function
organization, or translated code from AGPL or otherwise incompatible projects.
