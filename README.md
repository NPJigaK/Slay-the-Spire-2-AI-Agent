# Slay the Spire 2 AI Agent

Design-first project for a clean-room Slay the Spire 2 bridge that lets
external AI agents observe the current game state, list legal actions, choose an
opaque `action_id`, and execute it through a stable local protocol.

Current phase: design and protocol planning. Source packages are intentionally
not scaffolded yet.

## Architecture

The accepted architecture is REST-first and legal-action-first:

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

The bridge mod will not call LLM APIs or store provider credentials. MCP, CLI,
and skills are adapters over the canonical REST protocol, not the primary
protocol.

## Design Docs

- [ADR-0001: REST-first, legal-action-first architecture](docs/adr/0001-rest-first-legal-action-first-architecture.md)
- [Bridge design](docs/design/bridge-design.md)
- [Phase 0 existing bridge investigation](docs/investigations/phase-0-existing-bridges.md)
- [Error codes](docs/protocol/error-codes.md)
- [State model](docs/protocol/state-model.md)
- [Action model](docs/protocol/action-model.md)
- [Trace format](docs/protocol/trace-format.md)
- [Testing and conformance](docs/protocol/testing-conformance.md)

## Safety Defaults

The bridge is a local game-control API. Localhost is not treated as inherently
safe.

Release defaults:

- Bind to `127.0.0.1` only.
- Require Bearer token auth for protected endpoints.
- Keep `/health` unauthenticated but minimal.
- Disable wildcard CORS.
- Disable remote bind.
- Disable debug commands.
- Disable multiplayer actions in the MVP.

## MVP Scope

The MVP is combat-first and protocol-stability-first. A local external agent
should be able to:

- observe current singleplayer state;
- list executable legal actions;
- choose one opaque `action_id`;
- execute through `/act`;
- receive consistent `stale_state`, `invalid_action`, and busy errors;
- produce a replayable JSONL trace.

Non-combat screens may be represented and must fail safely. Full card reward,
map, event, rest, and shop progression is added after the initial combat loop
is stable.
