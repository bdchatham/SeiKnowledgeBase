# SeiKnowledgeBase

A structured knowledge base of Sei blockchain internals, built through agentic research and persisted as Claude-readable memories. The goal is to eliminate redundant deep-dive research by capturing verified, code-traced findings once and making them instantly available to future Claude sessions.

## Purpose

Sei's codebase spans multiple forks (sei-tendermint, sei-cosmos, sei-chain, sei-db, sei-iavl) with Sei-specific modifications that are not documented upstream. When Claude needs to reason about Sei node behavior — state sync, peer networking, upgrade mechanics, sidecar configuration — it currently has to trace through source code from scratch every time.

This repo solves that by maintaining a library of research memories organized by topic. Each memory is a self-contained document with enough context for Claude to understand a subsystem without reading the source.

## Structure

```
memories/
  blockchain/     # Core chain mechanics: consensus, block execution, state machine
  networking/     # P2P, peer discovery, seed nodes, address book, connection management
  operations/     # State sync, block sync, snapshots, pruning, DB backends
  upgrades/       # Upgrade handlers, migration mechanics, binary compatibility
  configuration/  # config.toml, app.toml, seid flags, environment variables
```

Each memory file uses this format:

```markdown
---
topic: <descriptive title>
sources: <list of repos/files/versions traced>
verified: <date of last verification against source code>
confidence: high | medium | low
---

<content>
```

- **topic**: What this memory covers, specific enough to judge relevance
- **sources**: Exact repos, file paths, and versions the findings were traced from. This is critical — it lets future sessions verify whether the memory is stale
- **verified**: When the source code was last read to confirm these findings
- **confidence**: `high` = traced through actual code with snippets, `medium` = traced but some inference, `low` = based on docs/behavior rather than code

## How to Use These Memories

**As a reader (Claude in another project):** Pull this repo and read the relevant memory files before doing Sei-related work. Check the `verified` date and `sources` — if the source files have changed significantly since verification, re-trace before relying on the memory.

**As a writer (RALPHY or similar research agent):** When researching a Sei topic:
1. Check if a memory already exists — update rather than duplicate
2. Always trace through actual source code, not just documentation
3. Include code snippets for critical logic (the "why" behind behavior)
4. Record the exact file paths and versions you read
5. Be explicit about what you verified vs. inferred

## Research Priorities

Foundational topics that should be covered first (in rough dependency order):

1. **Sei node configuration** — every parameter in config.toml and app.toml, what it controls, valid values, and interactions between parameters
2. **Execution modes** — how full node, archive node, validator, seed node, and sentry node modes differ in behavior and configuration
3. **State sync** — the complete code path from discovery through snapshot restore, including sei-tendermint-specific modifications
4. **Block sync** — fast sync mechanics, peer selection, block verification
5. **Peer networking** — P2P stack, peer exchange, seed nodes, persistent peers, private peers, address book management
6. **Upgrade mechanics** — how chain upgrades work end-to-end, binary compatibility, state migrations
7. **Genesis** — genesis file format, chain initialization, genesis ceremonies
8. **Pruning and DB backends** — SeiDB, IAVL, pruning strategies, storage implications
9. **EVM integration** — Sei's parallel EVM, pointer contracts, EVM/Cosmos interop
10. **Sidecar architecture** — sei-chain sidecar, oracle, price feeder interactions

## Memory Quality Standards

- No hand-waving. If you can't trace it through code, say so explicitly.
- Include the version/commit of the code you traced. Sei forks diverge from upstream — always specify the Sei fork, not upstream Tendermint/Cosmos docs.
- Prefer concrete code snippets over descriptions. "The default is 40 peers (see DefaultMaxNumInboundPeers in config.go:XXX)" is better than "The default peer count is relatively high."
- Call out Sei-specific deviations from upstream. If sei-tendermint does something different from CometBFT, that's the most valuable thing to capture.
