---
title: "Sei OCC (Optimistic Concurrency Control) Parallel Transaction Execution"
category: execution
tags: [occ, parallel, multiversion-store, scheduler, conflict-detection]
sources:
  sei-cosmos:
    version: v0.3.66
    files:
      - tasks/scheduler.go
      - tasks/scheduler_test.go
      - store/multiversion/store.go
      - store/multiversion/mvkv.go
      - store/multiversion/data_structures.go
      - store/multiversion/memiterator.go
      - store/multiversion/mergeiterator.go
      - store/multiversion/trackediterator.go
      - types/occ/types.go
      - types/tx_batch.go
      - types/tx_tracer.go
      - types/context.go
      - baseapp/abci.go
      - baseapp/baseapp.go
      - baseapp/options.go
      - baseapp/deliver_tx_batch_test.go
      - server/config/config.go
      - x/accesscontrol/README.md
      - x/accesscontrol/keeper/keeper.go
      - x/bank/keeper/keeper.go
      - x/bank/keeper/deferred_cache.go
  sei-chain:
    version: v0.0.38
    files:
      - app/app.go
      - app/abci.go
      - x/evm/state/statedb.go
  sei-tendermint:
    version: v0.6.4
    notes: "No OCC-specific code; parallelism is entirely in the ABCI app layer"
---

# Sei OCC: Optimistic Concurrency Control for Parallel Transaction Execution

## 1. High-Level Execution Model

Sei implements **Block-STM-style Optimistic Concurrency Control (OCC)** for parallel
transaction execution within a single block. The core idea:

1. All transactions in a block are executed **optimistically in parallel**, as if they
   were independent.
2. Each transaction reads/writes through a **multiversion store** that tracks per-transaction
   read sets and write sets.
3. After execution, a **validation phase** checks whether any transaction read a value
   that was subsequently changed by a lower-indexed transaction.
4. Conflicting transactions are **aborted and re-executed** with updated data.
5. The process repeats in execute-validate rounds until all transactions are validated.
6. If conflicts are too high (>= 10 iterations), execution **falls back to sequential**
   processing for remaining transactions.

This is an app-layer optimization. Tendermint/CometBFT is unaware of it -- the
parallelism lives entirely within the `FinalizeBlock` / `DeliverTx` path in sei-cosmos
and sei-chain.

## 2. Architecture Overview

```
FinalizeBlocker (sei-chain app/app.go)
  |
  v
ProcessBlock
  |-- BeginBlock (sequential)
  |-- DecodeTransactionsConcurrently (parallel goroutines)
  |-- PartitionPrioritizedTxs (oracle votes, dex register/unregister go first)
  |-- ExecuteTxsConcurrently (prioritized batch)
  |     |-- if OCC enabled -> ProcessTXsWithOCC
  |     |     |-- Build DeliverTxEntry[] with estimated writesets
  |     |     |-- BaseApp.DeliverTxBatch
  |     |     |     |-- tasks.Scheduler.ProcessAll   <-- THE OCC ENGINE
  |     |     |     |     |-- executeAll (parallel via worker pool)
  |     |     |     |     |-- validateAll (parallel)
  |     |     |     |     |-- repeat until allValidated or maxIterations
  |     |     |     |     |-- WriteLatestToStore (flush to parent)
  |     |-- if OCC disabled -> ProcessBlockSynchronous
  |-- WriteDeferredBalances (bank module deferred sends)
  |-- MidBlock
  |-- ExecuteTxsConcurrently (non-prioritized batch)
  |-- WriteDeferredBalances (again, for second batch)
  |-- EndBlock (sequential)
```

## 3. The Scheduler (`tasks/scheduler.go`)

The scheduler is the OCC orchestrator. It manages the execute-validate loop.

### Key Types

```go
type scheduler struct {
    deliverTx          func(ctx sdk.Context, req RequestDeliverTx, tx sdk.Tx, checksum [32]byte) ResponseDeliverTx
    workers            int
    multiVersionStores map[sdk.StoreKey]multiversion.MultiVersionStore
    allTasks           []*deliverTxTask
    executeCh          chan func()   // work queue for execution goroutines
    validateCh         chan func()   // work queue for validation goroutines
    synchronous        bool          // true when fallback to sequential
    maxIncarnation     int           // highest incarnation seen
}

type deliverTxTask struct {
    Status        status           // pending | executed | aborted | validated | waiting
    Incarnation   int              // how many times this tx has been (re-)executed
    AbsoluteIndex int              // position in the block's tx list
    Dependencies  map[int]struct{} // indices of txs this task depends on
    Abort         *occ.Abort       // details of why it aborted
    VersionStores map[StoreKey]*multiversion.VersionIndexedStore
    Response      *ResponseDeliverTx
    AbortCh       chan occ.Abort   // receives abort signal during execution
}
```

### Task Status State Machine

```
pending --> (execute) --> executed --> (validate: ok) --> validated
                     \                  |
                      \                 | (validate: conflict found)
                       \                v
                        --> aborted --> pending (reset + increment incarnation)
                                        |
                                        v
                                       waiting (if dependency not yet validated)
                                        |
                                        | (dependency validated)
                                        v
                                       pending (re-execute)
```

### ProcessAll: The Main Loop

```go
func (s *scheduler) ProcessAll(ctx, reqs) ([]ResponseDeliverTx, error) {
    // 1. Initialize multiversion stores (one per KV store key)
    s.tryInitMultiVersionStore(ctx)

    // 2. Convert requests to tasks (all start as statusPending)
    tasks, tasksMap := toTasks(reqs)

    // 3. Start worker pools
    //    - execution workers: min(workers, len(tasks)), default 20
    //    - validation workers: len(tasks) -- never block on validation
    start(workerCtx, s.executeCh, workers)
    start(workerCtx, s.validateCh, len(tasks))

    // 4. Execute-validate loop
    toExecute := tasks
    for !allValidated(tasks) {
        if iterations >= 10 {       // maximumIterations = 10
            s.synchronous = true    // fallback to sequential
            startIdx = findFirstNonValidated()
            toExecute = tasks[startIdx:]
        }

        executeAll(ctx, toExecute)  // parallel execution
        toExecute = validateAll(ctx, tasks)  // parallel validation, returns tasks to re-execute
        iterations++
    }

    // 5. Flush all validated writes to parent store
    for _, mv := range s.multiVersionStores {
        mv.WriteLatestToStore()
    }
}
```

### Execute Phase

For each task:

1. **Prepare**: Create a `VersionIndexedStore` per KV store key, wrapping the
   multiversion store. This store intercepts all Get/Set/Delete/Iterator operations.
   An abort channel is created to receive early-abort signals.

2. **Run**: Call `app.DeliverTx(ctx, req, tx, checksum)` -- the normal transaction
   execution path (ante handlers, message handlers, etc.), but the context's KV stores
   are replaced with `VersionIndexedStore` instances.

3. **Handle result**:
   - If abort channel has a message: the tx read an ESTIMATE value (a placeholder
     from an invalidated earlier tx). Mark task as `statusAborted`, record dependency,
     write ESTIMATE markers for this tx's writeset.
   - If no abort: mark task as `statusExecuted`, write the tx's actual writeset to
     the multiversion store.

### Validate Phase

Validation processes ALL tasks (not just recently executed ones), starting from the
first non-validated task. For each task, `shouldRerun` determines the outcome:

- **aborted/pending**: needs re-execution
- **executed/validated**: run conflict detection via `findConflicts` ->
  `multiVersionStore.ValidateTransactionState`
  - If **invalid** (read stale data): invalidate writeset, record dependencies.
    If dependencies are already validated, re-execute immediately; otherwise
    set status to `waiting`.
  - If **valid with no conflicts**: mark as `statusValidated`
  - If **valid with conflicts** (estimates present): will validate next round
- **waiting**: check if dependencies are now validated; if so, ready to re-execute

Tasks that need re-execution are collected, reset (status -> pending, incarnation++),
and returned for the next execute round.

## 4. The Multiversion Store (`store/multiversion/`)

### Store (`store.go`)

One `Store` per KV store key (e.g., bank, staking, auth each get their own). It maintains:

- **multiVersionMap**: `sync.Map[string]MultiVersionValue` -- maps each key to a
  B-tree of versioned values, indexed by transaction index.
- **txWritesetKeys**: `sync.Map[int][]string` -- which keys each tx wrote.
- **txReadSets**: `sync.Map[int]ReadSet` -- what each tx read and the values it saw.
- **txIterateSets**: `sync.Map[int]Iterateset` -- ranges each tx iterated over.
- **parentStore**: the actual underlying KV store.

### MultiVersionValue (`data_structures.go`)

Each key in the store has a B-tree (`google/btree`) of `valueItem` entries, sorted by
transaction index. Each entry stores:

```go
type valueItem struct {
    index       int     // transaction index that wrote this
    incarnation int     // which incarnation of the tx
    value       []byte  // the actual value (nil + !estimate = deleted)
    estimate    bool    // true if this is an ESTIMATE placeholder
}
```

**ESTIMATE** values are placeholders written when a transaction's writeset is
invalidated. They signal to later transactions "this key was written by tx N, but
we don't know the final value yet -- you must abort and wait."

Key operations:
- `GetLatestBeforeIndex(index)`: returns the most recent value written by any tx
  with index < the given index. This is how a tx "sees" writes from earlier txs.
- `Set(index, incarnation, value)`: write a value for a specific tx.
- `SetEstimate(index, incarnation)`: replace a value with an ESTIMATE marker.
- `Remove(index)`: delete a tx's entry entirely.

### VersionIndexedStore (`mvkv.go`)

This is what each transaction actually interacts with. It implements `types.KVStore`
and intercepts all operations:

**Get(key)**:
1. Check local writeset (this tx's own writes) -- return if found.
2. Check local readset (previously read) -- return cached value.
3. Query `multiVersionStore.GetLatestBeforeIndex(txIndex, key)`:
   - If result is an **ESTIMATE**: write abort to channel, panic with `occ.Abort`.
     The scheduler catches this panic via the abort channel.
   - If result is a real value or deletion: record in readset, return value.
4. If not in multiversion store at all: read from parent store, record in readset.

**Set(key, value)** / **Delete(key)**:
- Write to local writeset only. Values are flushed to the multiversion store after
  execution completes.

**Iterator(start, end)**:
- Creates a merge iterator combining:
  - Parent store iterator (the base KV store)
  - Memory iterator over multiversion store keys (from txs with lower indices)
  - Local writeset keys
- Wrapped in a `trackedIterator` that records all iterated keys into an
  `iterationTracker` for later validation.

### Conflict Detection: `ValidateTransactionState(index)`

Two checks are performed:

**1. Readset validation** (`checkReadsetAtIndex`):
For each key the tx read, check what the current latest value before this tx's index is:
- If the value is an **ESTIMATE**: record as conflict (the writing tx hasn't finished yet).
- If the value is **deleted** but tx read non-nil: conflict, invalid.
- If the value **differs** from what was read: conflict, invalid.
- If the value **matches**: valid for this key.

Returns `(valid bool, conflictIndices []int)`.

**2. Iterateset validation** (`checkIteratorAtIndex`):
For each iteration range the tx used, replay the iteration using current multiversion
store state and verify:
- Same keys appear in the same order.
- No new keys have appeared in the range.
- No keys have disappeared from the range.
- Early stop key is still correct.

Both checks must pass for the transaction to be considered valid.

## 5. Abort Mechanism

When a transaction reads an ESTIMATE value during execution:

```go
// In VersionIndexedStore.Get():
if mvsValue.IsEstimate() {
    abort := scheduler.NewEstimateAbort(mvsValue.Index())
    store.WriteAbort(abort)   // non-blocking send to abort channel
    panic(abort)              // immediately stop execution
}
```

The panic unwinds the stack. The scheduler catches it:

```go
// In scheduler.executeTask():
resp := s.deliverTx(task.Ctx, task.Request, task.SdkTx, task.Checksum)
close(task.AbortCh)
abort, ok := <-task.AbortCh
if ok {
    task.SetStatus(statusAborted)
    task.Abort = &abort
    task.AppendDependencies([]int{abort.DependentTxIdx})
    // Write estimates for this tx's writeset (signal to later txs)
    for _, v := range task.VersionStores {
        v.WriteEstimatesToMultiVersionStore()
    }
    return
}
```

The `Abort` struct carries:
```go
type Abort struct {
    DependentTxIdx int    // index of the tx whose estimate was read
    Err            error  // always ErrReadEstimate
}
```

## 6. Retry Strategy

- Each re-execution increments `task.Incarnation`.
- The task's old writeset is invalidated (replaced with ESTIMATE markers).
- The task's readset and iterateset are cleared.
- The task is re-prepared with fresh `VersionIndexedStore` instances.
- If `incarnation >= 10` (`maximumIterations`), the scheduler switches to
  **synchronous mode** (`s.synchronous = true`). In this mode, work items are
  executed inline instead of dispatched to the worker pool. The scheduler processes
  remaining tasks starting from the first non-validated one, executing them in order.

This guarantees forward progress: worst case, the block processes sequentially.

## 7. Block Execution Lifecycle Integration

```
FinalizeBlock
  |
  ProcessBlock(ctx, txs, req, lastCommit)
    |
    |-- ctx.WithIsOCCEnabled(app.OccEnabled())
    |
    |-- BeginBlock  (sequential, runs module begin-blockers)
    |
    |-- DecodeTransactionsConcurrently  (parallel goroutines, one per tx)
    |
    |-- PartitionPrioritizedTxs
    |     Splits into: prioritized (oracle votes, dex register/unregister)
    |                  and other (everything else)
    |
    |-- ExecuteTxsConcurrently(prioritizedTxs)  <-- OCC batch 1
    |     |-- ProcessTXsWithOCC
    |           |-- Generate estimated writesets (parallel, via access control keeper)
    |           |-- BaseApp.DeliverTxBatch -> Scheduler.ProcessAll
    |
    |-- BankKeeper.WriteDeferredBalances  (flush deferred bank transfers)
    |
    |-- MidBlock  (module mid-blockers, e.g. oracle)
    |
    |-- ExecuteTxsConcurrently(otherTxs)  <-- OCC batch 2
    |     |-- ProcessTXsWithOCC (same flow)
    |
    |-- BankKeeper.WriteDeferredBalances  (flush second batch)
    |
    |-- EndBlock  (sequential, runs module end-blockers)
```

Key points:
- **Two OCC batches per block**: prioritized txs run first, then everything else.
  This ensures oracle votes and DEX registration are processed before regular trades.
- **BeginBlock and EndBlock are sequential** -- OCC only applies to the DeliverTx phase.
- **MidBlock** runs between the two batches, currently used for oracle price aggregation.
- **Deferred bank balances** are flushed after each batch -- this is how the bank module
  avoids conflicts on module account balances during parallel execution.

## 8. Transaction Ordering Guarantees

- **Original block ordering is preserved in the final result.** Each task has an
  `AbsoluteIndex` from the original block position. The multiversion store uses this
  index for all versioning -- `GetLatestBeforeIndex(index)` ensures a tx at index N
  only sees writes from txs 0..N-1.
- **Within the OCC engine, execution order is non-deterministic** -- any tx can run on
  any worker at any time. But the validation phase ensures the final committed state
  is identical to sequential execution in original order.
- **The two-batch split (prioritized vs other)** introduces a sequencing barrier: all
  prioritized txs are fully processed before any non-prioritized tx begins.

## 9. EVM Transaction Interaction

EVM transactions participate in OCC the same way as Cosmos transactions:

- The EVM `StateDB` (`x/evm/state/statedb.go`) reads `ctx.TxIndex()` to get its
  transaction position, which is set by the scheduler's `prepareTask`.
- EVM state reads/writes go through the same Cosmos KV store layer, which is wrapped
  by `VersionIndexedStore` during OCC execution.
- Each EVM tx gets a unique coinbase address derived from its `TxIndex`.
- The `TxTracer` interface allows EVM tracing to be reset on re-execution and committed
  on final validation.

EVM and Cosmos txs can conflict with each other if they touch the same KV store keys
(e.g., bank balances, EVM state slots that map to Cosmos storage).

## 10. Performance Configuration

| Parameter | Flag | Default | Description |
|-----------|------|---------|-------------|
| Concurrency Workers | `--concurrency-workers` | 20 | Number of goroutines in the execution worker pool |
| OCC Enabled | `--occ-enabled` | (app-configured) | Whether to use OCC or fall back to sequential |

The scheduler also creates `len(tasks)` validation workers to avoid blocking during
the validation phase.

If `workers < 1` or `workers > len(tasks)`, the scheduler uses `len(tasks)` as the
worker count. This means for small blocks, every tx gets its own goroutine.

## 11. Estimated Writesets (Optimization)

Before OCC execution, the access control keeper generates **estimated writesets** for
each transaction based on its message types and the registered access control dependency
mappings. These are pre-populated as ESTIMATE markers in the multiversion store via
`PrefillEstimates`.

The idea: if tx N is known to write key K, and tx M (M > N) tries to read K before
tx N has executed, the ESTIMATE marker immediately triggers an abort for tx M instead
of letting it read stale data and discover the conflict later during validation.

**Note**: As of v0.3.66, this optimization is **disabled** in the scheduler code:
```go
// This "optimization" path is being disabled because we don't have a strong reason
// to have it given that it
// s.PrefillEstimates(reqs)
```
The estimated writesets are still generated but not pre-filled.

## 12. Deferred Bank Module Cache

The bank module uses a `DeferredCache` to avoid conflicts on hot module account
balances. Instead of directly crediting/debiting module accounts during tx execution
(which would cause every fee-paying tx to conflict on the fee collector balance),
balance changes are written to a per-tx-indexed deferred store:

```
prefix: DeferredCachePrefix / moduleAddr / txIndex / denom -> balance
```

After each OCC batch completes, `WriteDeferredBalances` aggregates all deferred
balances per module, sorts them for determinism, and applies them to the actual
bank store in a single sequential pass. This eliminates a major source of false
conflicts.

## 13. Fallback to Sequential

The system has multiple fallback paths:

1. **OCC disabled** (`--occ-enabled=false`): `ExecuteTxsConcurrently` calls
   `ProcessBlockSynchronous` directly.
2. **Max iterations exceeded** (>= 10 rounds): The scheduler sets `s.synchronous = true`
   and processes remaining tasks inline, starting from the first non-validated task.
3. **Legacy DAG-based execution** (`BuildDependenciesAndRunTxs`): Deprecated, always
   falls through to synchronous. Left in codebase but commented out.
4. **Pre-OCC concurrent execution** (`ProcessTxs` / `ProcessBlockConcurrent`): Uses
   a dependency DAG with completion signals. Still present but only used if OCC is
   disabled. Has its own fallback: if any tx gets `ErrInvalidConcurrencyExecution`,
   the entire block is re-executed synchronously.

## 14. Metrics and Observability

The scheduler emits telemetry:
- `scheduler.retries`: total number of tx re-executions across all iterations.
- `scheduler.incarnations`: the maximum incarnation seen (measures worst-case contention).
- Log line per block: `"occ scheduler" height=X txs=N latency_ms=T retries=R maxIncarnation=I iterations=J sync=bool workers=W`

The `TxTracer` interface provides hooks for EVM-level tracing that correctly handles
OCC re-execution:
- `Reset()`: called before each re-execution, discarding previous trace data.
- `Commit()`: called once when the tx is finally validated.
- `InjectInContext()`: adds the tracer to the execution context.

## 15. Summary: What Makes This Work

1. **Multiversion store with B-tree per key**: O(log n) lookups for "latest value before
   index I", enabling snapshot isolation per tx.
2. **ESTIMATE markers**: Enable early abort when a dependent tx hasn't finished yet,
   avoiding wasted execution.
3. **Read/write/iterate set tracking**: Comprehensive conflict detection covering point
   reads, deletions, and range iterations.
4. **Incarnation-aware validation**: Each re-execution produces a new incarnation,
   and the multiversion store tracks which incarnation wrote each value.
5. **Graceful degradation**: Sequential fallback after 10 iterations prevents livelock
   under pathological contention.
6. **Deferred bank cache**: Eliminates the most common source of false conflicts
   (fee collector module account).
7. **Two-phase block execution**: Prioritized txs (oracle, DEX admin) run first,
   reducing cross-concern conflicts.
