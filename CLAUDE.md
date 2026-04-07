# SeiKnowledgeBase

Structured knowledge base of Sei blockchain internals, built through Tide research agents tracing source code. Eliminates redundant deep-dive research by capturing verified findings as Claude-consumable memories.

## Structure

```
memories/
  node-configuration/   # How Sei nodes are configured and run
```

Categories are added as research covers new ground. Keep them minimal — one category per coherent domain, not one per subtopic.

### node-configuration

Research topics within this category:

- **Node types** — full node, archive, validator, seed, sentry. How configurations differ between them and what makes each mode distinct.
- **sei-config package** — the `sei-config` library itself. How it defines, loads, and exposes configuration to the node binary.
- **seid configuration consumption** — how `seid` reads config.toml, app.toml, and sei-config values at startup. The chain from config file to runtime behavior.
- **State sync** — the state sync code path end-to-end, trust parameters, snapshot mechanics.

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
