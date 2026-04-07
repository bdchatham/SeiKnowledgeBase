---
title: "Giga Executor & Autobahn -- WIP Status"
domain: sei-execution
confidence: medium
date: 2026-04-07
source_versions:
  sei-chain: v0.0.38
  sei-cosmos: v0.3.66
  sei-tendermint: v0.6.4
  sei-config: v0.0.9-0.20260327015454-7cf35ff77daa
---

# Giga Executor & Autobahn -- Current State

## Executive Summary

**Giga Executor** exists as a configuration stub in sei-config but has NO implementation
in sei-chain, sei-cosmos, or sei-tendermint at these versions. It is a planned replacement
or evolution of the existing OCC parallel execution system.

**Autobahn** has ZERO references anywhere in the codebase -- no code, no config, no
comments. It either lives in a different repo, uses a different internal name, or has
not yet landed in any of the four Sei modules examined.

## What Is Giga? (Inferred from Config + Architecture)

Giga Executor appears to be a next-generation parallel transaction execution engine
intended to replace or subsume the current OCC-based executor. Evidence:

1. **Config structure** (`sei-config` `GigaExecutorConfig`):
   - `enabled` (bool) -- master toggle for the Giga engine
   - `occ_enabled` (bool) -- toggle for OCC *within* Giga

2. **Config enrichment description**: "Enable the Giga parallel execution engine"
   and "Enable OCC within the Giga executor" -- this framing implies Giga is a
   broader execution engine that can optionally use OCC as one strategy.

3. **Defaults**: GigaExecutor is NOT included in `baseDefaults()`, so both fields
   default to `false`. This confirms it is not yet production-ready.

4. **No consumer**: sei-chain v0.0.38 does not reference `GigaExecutor` anywhere --
   not in `app.go`, not in `cmd/seid/cmd/root.go`, not in any controller or keeper.
   The config field exists but is never read.

### Relationship to Current OCC

The existing parallel execution system is fully operational and uses:

- **`chain.occ_enabled`** (bool, defaults `true`) -- enables OCC at the app level
- **`chain.concurrency_workers`** (int, defaults `runtime.NumCPU() * 2`, clamped 10-128)

The implication of `giga_executor.occ_enabled` being a *separate* flag from `chain.occ_enabled`
suggests Giga may introduce alternative parallelization strategies beyond OCC
(speculative: pipelining, sharding, or static scheduling), with OCC as one option
that can be toggled independently within the Giga framework.

## Current OCC Parallel Execution (What Giga Will Replace)

The production execution pipeline as of these versions:

### Flow

1. **Tendermint** calls `FinalizeBlock` on sei-chain's `App`
2. `App.ProcessBlock()` sets `ctx.WithIsOCCEnabled(app.OccEnabled())`
3. Transactions are decoded concurrently (`DecodeTransactionsConcurrently`)
4. Txs are partitioned into prioritized + other (`PartitionPrioritizedTxs`)
5. Each partition goes through `ExecuteTxsConcurrently`:
   - If OCC enabled: `ProcessTXsWithOCC` -> `DeliverTxBatch` -> `tasks.Scheduler.ProcessAll`
   - If OCC disabled: `ProcessBlockSynchronous` (sequential fallback)
6. Between partitions, deferred bank writes are flushed and MidBlock runs

### OCC Scheduler (`sei-cosmos/tasks/scheduler.go`)

The OCC implementation is a multi-version concurrency control (MVCC) scheduler:

- Creates `MultiVersionStore` per KV store key (wraps the underlying store)
- Executes all transactions concurrently via worker goroutines
- Each tx gets a `VersionIndexedStore` that tracks reads/writes
- After execution, validates by checking read-sets against concurrent writes
- Conflicting transactions are re-executed (with dependency tracking)
- Falls back to synchronous after `maximumIterations` (10) retries
- Supports estimated writesets for prefill optimization (currently disabled)

Key types:
- `tasks.Scheduler` interface with `ProcessAll(ctx, reqs) ([]ResponseDeliverTx, error)`
- `deliverTxTask` -- per-tx state machine (pending -> executed -> validated, or aborted)
- `multiversion.MultiVersionStore` -- MVCC storage layer
- `occ.Abort` -- conflict signaling between tasks

### Access Control / Dependency Estimation

- `aclmapping/` contains per-module access control mappings (bank, dex, evm, oracle, staking, tokenfactory, wasm)
- `AccessControlKeeper.GenerateEstimatedWritesets` produces prefill estimates for OCC
- `BuildDependencyDag` (deprecated, commented out) was the older DAG-based approach
- `parallelization/` contains Rust test contracts for bank/staking/wasm parallelism testing

### Deprecated Path

`BuildDependenciesAndRunTxs` is marked deprecated. The DAG-based concurrent execution
(`ProcessBlockConcurrent`, `ProcessTxs`) is commented out and falls through to synchronous.
OCC has fully replaced the DAG approach.

## What Is Autobahn?

**No code exists for Autobahn** in any of the four repositories at these versions.

Searches performed:
- Case-insensitive grep for `autobahn` across sei-chain, sei-cosmos, sei-tendermint, sei-config
- All returned zero results

Autobahn may be:
- A marketing/product name for Giga or the overall parallel execution initiative
- Code that lives in a separate repository not included in these dependencies
- A feature that has not yet been committed to any of these modules
- An internal project name used in design documents but not yet in code

**Confidence: low** -- cannot determine what Autobahn is from the available source code.

## Configuration Parameters Summary

### Production (active)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `chain.concurrency_workers` | int | `NumCPU*2` (10-128) | Worker count for parallel tx execution |
| `chain.occ_enabled` | bool | `true` | Enable OCC for transaction processing |

### Giga (config exists, no implementation)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `giga_executor.enabled` | bool | `false` | Master toggle for Giga engine |
| `giga_executor.occ_enabled` | bool | `false` | Enable OCC within Giga |

## What Is Clearly WIP/Incomplete

1. **GigaExecutor config is defined but never consumed** -- the struct exists in sei-config
   but sei-chain does not read it. No conditional logic references it.

2. **No Giga execution engine code exists** -- there is no alternative scheduler,
   no new execution pipeline, no Giga-specific types or interfaces in any repo.

3. **Estimated writeset prefill is disabled** -- the `PrefillEstimates` method exists
   in the scheduler but the call is commented out with a note that there's no strong
   reason to use it. This may be a Giga prerequisite.

4. **DAG-based execution is fully deprecated** -- `BuildDependenciesAndRunTxs` is
   commented out and falls through to synchronous. This was the pre-OCC approach.

5. **Autobahn has no code presence** -- completely absent from all four repositories.

## Key File Locations

- Config definition: `sei-config/config.go` (lines 448-451, `GigaExecutorConfig`)
- Config enrichments: `sei-config/enrichments.go` (lines 362-369)
- Legacy config mapping: `sei-config/legacy.go` (lines 200, 320-323, 628-631, 940-943)
- OCC scheduler: `sei-cosmos/tasks/scheduler.go`
- Multi-version store: `sei-cosmos/store/multiversion/`
- OCC abort types: `sei-cosmos/types/occ/types.go`
- App execution pipeline: `sei-chain/app/app.go` (lines 1413-1565)
- ABCI wiring: `sei-chain/app/abci.go`
- Batch delivery: `sei-cosmos/baseapp/abci.go` (lines 257-277)
- BaseApp OCC flags: `sei-cosmos/baseapp/baseapp.go`, `sei-cosmos/baseapp/options.go`
