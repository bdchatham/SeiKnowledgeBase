# SeiKnowledgeBase

Structured knowledge base of Sei blockchain internals, built through Tide research agents tracing source code. Eliminates redundant deep-dive research by capturing verified findings as Claude-consumable memories.

## Structure

```
memories/
  node-configuration/   # How Sei nodes are configured and run
  sei-consensus/        # Tendermint fork, block production, consensus modifications
  sei-execution/        # Transaction execution: OCC, Giga, Autobahn, parallelism
  sei-storage/          # SeiDB, MemIAVL, state-commit, state-store, pruning
  sei-evm/              # Parallel EVM, pointer contracts, EVM/Cosmos interop
```

Categories are added as research covers new ground. Keep them minimal — one category per coherent domain, not one per subtopic.

### node-configuration

- **Node types** — full node, archive, validator, seed, sentry. Configuration differences.
- **sei-config package** — the config library internals, loading, validation, ConfigIntent pipeline.
- **seid configuration consumption** — startup path from `seid start` through Viper to runtime.
- **State sync** — state sync code path end-to-end, trust parameters, snapshot mechanics.

### sei-consensus

- **sei-tendermint fork overview** — what Sei changed from upstream CometBFT and why. The delta.
- **Block production** — proposal, prevote, precommit flow with Sei modifications.
- **P2P layer** — peer management, gossip, Sei-specific networking changes.
- **DB sync** — Sei's alternative to state sync for bootstrapping nodes.

### sei-execution

- **OCC (Optimistic Concurrency Control)** — parallel tx execution, conflict detection, abort/retry.
- **Giga executor** — next-gen execution engine (WIP). Architecture and current state.
- **Autobahn** — high-throughput pipeline (WIP). Architecture and current state.
- **Transaction lifecycle** — from mempool through execution to commit.

### sei-storage

- **SeiDB architecture** — state-commit (MemIAVL) vs state-store (PebbleDB), why two layers.
- **MemIAVL** — Sei's in-memory IAVL fork, how it differs from standard IAVL.
- **Pruning and compaction** — how state is pruned across both storage layers.
- **Snapshot mechanics** — how snapshots are created and consumed at the storage level.

### sei-evm

- **Parallel EVM** — how Sei runs EVM transactions in parallel with Cosmos txs.
- **Pointer contracts** — bridging EVM and Cosmos address spaces.
- **EVM RPC** — Sei's Ethereum-compatible JSON-RPC layer.

## Memory Format

```markdown
---
topic: <descriptive title>
sources:
  - repo: <e.g., sei-tendermint>
    version: <e.g., v0.6.4>
    files:
      - <relative path within the module>
verified: <YYYY-MM-DD>
confidence: high | medium | low
---

<content>
```

- **sources**: Exact repos, file paths, and versions traced. Lets future sessions verify staleness.
- **verified**: Date the source code was last read to confirm findings.
- **confidence**: `high` = code-traced with snippets, `medium` = traced with inference, `low` = docs/behavior only.

## Quality Standards

- Trace actual source code, not upstream Tendermint/Cosmos docs. Sei forks diverge.
- Include file paths and code snippets for key findings.
- Call out Sei-specific deviations from upstream explicitly.
- Don't duplicate — update existing memories when new information surfaces.
- Be dense, not verbose. Code snippets > prose.
