# Phase 0 Existing Bridge Investigation

This document records observations for the Phase 0 investigation spike. It is
not an implementation source. Clean-room implementation must not copy code,
line-by-line structure, class/function organization, or translated code from
existing bridge projects.

## Scope

Compare existing Slay the Spire 2 bridge and agent projects as behavior,
protocol, and design references:

- STS2MCP: https://github.com/Gennadiyev/STS2MCP
- STS2-Agent: https://github.com/CharTyr/STS2-Agent
- sts2-mcp: https://github.com/Yidhar/sts2-mcp
- StS1 CommunicationMod: https://github.com/ForgottenArbiter/CommunicationMod
- StS1 AI reference: https://github.com/tegnike/slay-the-spire-ai

## Evaluation Axes

```text
API surface
state schema
legal action model
action_id execution
stale-state guard
logging / trace
timeout handling
event stream
batching
cross-platform support
license
MCP adapter shape
testability
Early Access resilience
```

## Initial Observed Facts

STS2MCP:

- Provides a localhost REST API and optional MCP server.
- Separates singleplayer and multiplayer endpoint areas.
- Uses a local HTTP server inside the mod.
- Uses action validation before executing game actions.
- Is MIT licensed.

STS2-Agent:

- Exposes local HTTP API and MCP wrapping.
- Documents live state, legal actions, SSE events, and training-oriented
  payloads.
- Is AGPL-3.0-only, so implementation code must not be copied into this
  project.

Yidhar/sts2-mcp:

- Documents a C# bridge mod plus Node.js MCP server stack.
- Documents legal action exposure, unique action IDs, and `state_version`
  guarding.
- README indicates Windows-only prerequisites, so cross-platform suitability
  needs separate validation.

StS1 CommunicationMod:

- Demonstrates a state JSON plus command loop for an external process.
- Uses stdin/stdout rather than REST.
- Useful as a protocol reference, but the StS1 Java modding stack does not
  transfer directly to StS2 Godot/C#.

## Design Ideas To Consider

- REST API as the primary interface.
- MCP as an adapter rather than the primary protocol.
- `/actions` returns legal executable actions.
- `/act` executes an opaque action token.
- `state_id` or `state_version` guards stale actions.
- Main-thread execution for game operations.
- JSONL traces for state, actions, decisions, results, and errors.
- Event stream or wait-until-actionable as a later feature.

## Ideas To Reject Or Defer

- MCP-first core protocol.
- Free-form action commands in the primary loop.
- Copying implementation code from existing bridges.
- Multiplayer control in the MVP.
- Adapter-level batching before single-action correctness is stable.
- In-process LLM calls from the bridge mod.

## License Notes

- STS2MCP is MIT licensed at the time of initial investigation.
- STS2-Agent is AGPL-3.0-only at the time of initial investigation.
- sts2-mcp is reported as MIT in initial research and should be rechecked
  during the formal spike.

License status must be verified from repository files during Phase 0 before
any code-level comparison is used.

## Do-not-copy Notes

Allowed:

- Describe observed API behavior.
- Compare feature coverage.
- Record protocol-level ideas.
- Cite source links.

Not allowed:

- Copy implementation code.
- Translate source code line-by-line.
- Preserve class/function structure from incompatible licenses.
- Import source files or generated artifacts from incompatible projects.

## Investigation Output

Phase 0 is complete when this document includes:

- black-box behavior notes;
- source-reading notes separated from behavior notes;
- adopted design observations;
- rejected design observations;
- unresolved risks;
- license and do-not-copy notes;
- links to observed source material.
