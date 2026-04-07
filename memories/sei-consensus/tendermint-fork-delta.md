---
topic: "sei-tendermint fork: complete delta from upstream CometBFT"
sources:
  - repo: sei-tendermint
    version: v0.6.4
    files:
      - FORKED_CHANGELOG.md
      - go.mod
      - abci/types/application.go
      - abci/types/types.go
      - abci/types/types.pb.go
      - abci/types/messages.go
      - proto/tendermint/abci/types.proto
      - proto/tendermint/types/types.proto
      - proto/tendermint/dbsync/types.proto
      - internal/consensus/state.go
      - internal/consensus/reactor.go
      - internal/consensus/wal.go
      - internal/consensus/metrics.go
      - internal/mempool/mempool.go
      - internal/mempool/tx.go
      - internal/mempool/priority_queue.go
      - internal/mempool/types.go
      - internal/mempool/cache.go
      - internal/mempool/reactor.go
      - internal/dbsync/syncer.go
      - internal/dbsync/reactor.go
      - internal/dbsync/snapshot.go
      - internal/state/execution.go
      - internal/statesync/reactor.go
      - internal/blocksync/reactor.go
      - internal/p2p/pex/reactor.go
      - internal/p2p/router.go
      - node/node.go
      - node/public.go
      - node/setup.go
      - config/config.go
      - config/toml.go
      - types/proposal.go
      - types/tx.go
      - types/mempool.go
      - types/block.go
      - types/params.go
      - types/part_set.go
      - utils/slice.go
      - export/export.go
      - cmd/tendermint/commands/snapshot.go
verified: 2026-04-07
confidence: high
---

# sei-tendermint Fork: Complete Delta from Upstream Tendermint v0.35

sei-tendermint is forked from Tendermint v0.35.x (the last pre-CometBFT release). The module path
remains `github.com/tendermint/tendermint` but the repo lives at `github.com/sei-protocol/sei-tendermint`.

The FORKED_CHANGELOG.md (dated June 13, 2023) lists the high-level changes. This document exhaustively
catalogs every divergence found in the source code.

---

## 1. ABCI Changes

### 1.1 New ABCI Method: `LoadLatest`

**Files:** `abci/types/application.go`, `proto/tendermint/abci/types.proto`

Sei adds a new ABCI method to the `Application` interface:

```go
LoadLatest(context.Context, *RequestLoadLatest) (*ResponseLoadLatest, error)
```

- Proto field numbers: `Request.load_latest = 23`, `Response.load_latest = 24`
- `RequestLoadLatest` and `ResponseLoadLatest` are both empty messages
- **Purpose:** Called after DB sync completes to tell the application to reload its state from the
  freshly-synced database files. The node controller calls `client.LoadLatest()` in the post-DB-sync
  hook before switching to block sync.

### 1.2 `CheckTx` Returns `ResponseCheckTxV2` (not `ResponseCheckTx`)

**Files:** `abci/types/application.go`, `abci/types/types.go`

The `Application.CheckTx` signature is changed:

```go
// Upstream:
CheckTx(context.Context, *RequestCheckTx) (*ResponseCheckTx, error)
// Sei:
CheckTx(context.Context, *RequestCheckTx) (*ResponseCheckTxV2, error)
```

`ResponseCheckTxV2` wraps the standard `ResponseCheckTx` with Sei-specific fields:

```go
type ResponseCheckTxV2 struct {
    *ResponseCheckTx
    IsPendingTransaction bool
    Checker              PendingTxChecker  // callback polled to determine if pending tx is ready
    ExpireTxHandler      ExpireTxHandler

    // EVM-specific prioritization helpers
    EVMNonce         uint64
    EVMSenderAddress string
    IsEVM            bool
}
```

These are non-protobuf fields, so they only work with in-process (local) ABCI clients.

### 1.3 New Proto Message: `EvmTxInfo`

**Files:** `proto/tendermint/abci/types.proto`

```protobuf
message EvmTxInfo {
  string senderAddress = 1;
  uint64 nonce         = 2;
  string txHash        = 3;
  string vmError       = 4;
}
```

Added as field 9 on both `ResponseDeliverTx` and `ExecTxResult`. Used by the mempool during `Update()`
to track EVM sender/nonce for correct nonce-ordered queuing.

### 1.4 Pending Transaction Support

**Types:** `PendingTxChecker`, `PendingTxCheckerResponse` (Accepted/Rejected/Pending), `ExpireTxHandler`

These types allow the application to return a "pending" status from CheckTx. The transaction is held
in a separate pending store and periodically re-evaluated via the `Checker` callback. When the checker
returns `Accepted`, the tx is promoted into the main mempool priority queue.

---

## 2. Consensus Changes

### 2.1 TxKey-Based Proposal Gossip (`gossip-tx-key-only`)

**Files:** `internal/consensus/state.go`, `types/proposal.go`, `types/mempool.go`, `proto/tendermint/types/types.proto`

This is the single largest consensus-layer change. Instead of gossiping full block data in proposals,
Sei can gossip only the SHA-256 hashes (TxKeys) of the transactions.

**Proto additions:**
```protobuf
message TxKey {
  bytes tx_key = 1;
}
// In Proposal message:
repeated TxKey tx_keys = 8;
```

**Go types:**
```go
type TxKey [sha256.Size]byte  // in types/mempool.go

// Proposal struct gains:
TxKeys          []TxKey
Header          Header
LastCommit      *Commit
Evidence        EvidenceList
ProposerAddress Address
```

The Proposal now carries the full header, last commit, evidence, and proposer address (in addition to
the standard BlockID), so that non-proposer validators can reconstruct the block from their local
mempool.

**Behavior when `gossip-tx-key-only = true` (default: true):**

1. **Proposer** creates the block normally via `createProposalBlock()`, then includes `block.GetTxKeys()`
   in the proposal message.
2. **Non-proposer validators** receive the proposal and immediately attempt to build the block from
   their local mempool via `buildProposalBlock()`:
   - Calls `blockExec.SafeGetTxsByKeys(txKeys)` to fetch transactions by hash from the mempool
   - If any transactions are missing, the block cannot be built (returns nil)
   - Missing txs are tracked via the `ProposalMissingTxs` metric
3. **Fallback:** If a validator cannot build the block from TxKeys, it waits for full block parts
   (traditional gossip). When block parts arrive, it verifies the hash proof and uses them.
4. **Prevote behavior:** In `defaultDoPrevote()`, if `GossipTransactionKeyOnly` is true and the
   proposal block is nil, the validator attempts to build from TxKeys or falls back to block parts.
   If neither succeeds, it prevotes nil.

**Performance benefit:** Proposals are tiny (just hashes), so proposal propagation is near-instant.
Validators already have most transactions from regular mempool gossip.

**Config:** `[consensus] gossip-tx-key-only = true` (default true, disabled in test config)

### 2.2 Proactive Block Building

When a non-proposer receives a proposal with TxKeys, it immediately tries to build the block
(`tryCreateProposalBlock`) without waiting for block parts. This is done in the `ProposalMessage`
handler inside `handleMsg()`, enabling validators to have the complete block before the prevote
timeout fires.

### 2.3 Block Part Size Increase

**File:** `types/params.go`

```go
BlockPartSizeBytes uint32 = 1048576 // 1MB (upstream: 65536 = 64KB)
```

Block parts are 16x larger than upstream (1MB vs 64KB), reducing the number of parts needed for
large blocks.

### 2.4 WAL Message Size Increase

**File:** `internal/consensus/reactor.go`

```go
maxMsgSize = 4194304 // 4MB (upstream: 1048576 = 1MB)
```

The consensus reactor message size limit is 4MB to accommodate larger block parts.

### 2.5 Fsync Optimization

**File:** `internal/consensus/state.go`

The `handleMsg()` function takes a `fsyncUponCompletion bool` parameter. WAL fsync is deferred
and batched rather than done synchronously on every message, reducing disk I/O during consensus.

### 2.6 OpenTelemetry Tracing

**File:** `internal/consensus/state.go`

The consensus state machine has extensive OpenTelemetry instrumentation:
- A `tracer` field (`otrace.Tracer`) is initialized from `trace.TracerProviderOption` passed at construction
- Spans are created for proposal handling, block part handling, vote handling, new rounds, prevote,
  precommit, commit, and finalize
- A per-height span (`heightSpan`) tracks the complete lifecycle of a height
- The tracing context is maintained in `tracingCtx` field

### 2.7 Additional Consensus Metrics

**File:** `internal/consensus/metrics.go`

New metrics added:
- `ProposalMissingTxs` (Gauge) -- number of missing txs when building from TxKeys
- `ProposalCreateCount` (Counter) -- number of proposal blocks created
- `StepLatency` (Gauge, labeled by step) -- time spent in each consensus step
- `MarkBlockGossipStarted()`/`MarkBlockGossipComplete()` -- block gossip timing

---

## 3. Mempool Changes

### 3.1 EVM-Aware Nonce Ordering

**Files:** `internal/mempool/priority_queue.go`, `internal/mempool/tx.go`

The priority queue has a dedicated EVM transaction queue:

```go
type TxPriorityQueue struct {
    txs      []*WrappedTx              // priority heap (standard + EVM heads)
    evmQueue map[string][]*WrappedTx   // per-address, sorted by nonce
}
```

**Invariants:**
1. No duplicate nonce in the same address queue
2. No nonce gap in the same address queue
3. Only the head (lowest nonce) of each address queue is in the main heap

**WrappedTx gains EVM fields:**
```go
evmAddress string
evmNonce   uint64
isEVM      bool
```

The `IsBefore()` method compares by EVM nonce for ordering within an address.

**Nonce replacement:** `tryReplacementUnsafe()` allows a higher-priority tx with the same nonce to
replace an existing one (EVM-style nonce replacement).

### 3.2 Pending Transaction Store

**Files:** `internal/mempool/tx.go`, `internal/mempool/mempool.go`

A separate `PendingTxs` store holds transactions whose `IsPendingTransaction` flag was set during
CheckTx. These are periodically evaluated via their `Checker` callback:

- `EvaluatePendingTransactions()` polls all pending txs and moves accepted ones to the main pool
- `PurgeExpired()` removes pending txs that exceed TTL
- Pending txs are included in `ReapMaxTxs()` if the main pool is short
- Config: `pending-size` (default 5000), `max-pending-txs-bytes` (default 1GB), `pending-ttl-duration`, `pending-ttl-num-blocks`

### 3.3 Estimated Gas / Triple Gas Constraint

**File:** `internal/mempool/mempool.go`

`ReapMaxBytesMaxGas()` accepts three constraints instead of two:
```go
func (txmp *TxMempool) ReapMaxBytesMaxGas(maxBytes, maxGasWanted, maxGasEstimated int64) types.Txs
```

For EVM transactions, `estimatedGas` (actual gas usage estimate) is used instead of `gasWanted`
when checking against `maxGasEstimated`. This prevents over-reserving gas for EVM txs that specify
high gas limits but actually use much less.

### 3.4 TxNotifyThreshold

**File:** `config/config.go`

Config field `tx-notify-threshold` (default 0): If non-zero, the mempool will not reap transactions
for block proposals until the mempool contains at least this many transactions. This prevents
proposing small blocks when the mempool is sparsely populated.

### 3.5 Duplicate Transaction Cache (TTL-based)

**File:** `internal/mempool/cache.go`

A `DuplicateTxCache` (backed by `github.com/patrickmn/go-cache`) tracks how many times each transaction
hash has been seen within a TTL window. Used for metrics reporting on duplicate transaction pressure.

Config: `duplicate-txs-cache-size` (default 100000)

### 3.6 CheckTx Error Blacklisting

**File:** `config/config.go`

Config fields: `check-tx-error-blacklist-enabled`, `check-tx-error-threshold`. If a peer sends more
than `threshold` transactions that fail CheckTx, the peer is blacklisted.

### 3.7 Mempool Reactor: ReadyToStart Gate

**File:** `internal/mempool/reactor.go`

The mempool reactor waits on a `readyToStart` channel before processing peer messages. This is used
during DB sync to prevent the mempool from processing transactions before the database is ready.
`MarkReadyToStart()` is called after DB sync completes.

### 3.8 Mempool Interface: TxKey-Based Methods

**File:** `internal/mempool/types.go`

New methods on the `Mempool` interface:
```go
HasTx(txKey types.TxKey) bool
GetTxsForKeys(txKeys []types.TxKey) types.Txs
SafeGetTxsForKeys(txKeys []types.TxKey) (types.Txs, []types.TxKey)
TxStore() *TxStore
```

These support the TxKey-based proposal building mechanism.

### 3.9 SHA-256 Truncated Cache Keys

**File:** `internal/mempool/mempool.go`

Cache keys use SHA-256 truncated to 128 bits (16 bytes) instead of full 32-byte hashes, reducing
memory overhead while maintaining negligible collision probability at production tx rates.

---

## 4. DB Sync (Entirely New Subsystem)

**Files:** `internal/dbsync/` (new package), `proto/tendermint/dbsync/types.proto`, `cmd/tendermint/commands/snapshot.go`

DB sync is a Sei-specific mechanism for bootstrapping a node by directly copying the application
database files from a peer, bypassing state sync's chunk-based ABCI approach.

### 4.1 Architecture

DB sync operates on 4 custom P2P channels:
- `MetadataChannel` (0x70) -- exchange snapshot metadata (filenames, checksums, height)
- `FileChannel` (0x71) -- exchange raw database files
- `LightBlockChannel` (0x72) -- exchange light blocks for verification
- `ParamsChannel` (0x73) -- exchange consensus params

### 4.2 Protocol Flow

1. Syncing node broadcasts `MetadataRequest` to all peers
2. Serving peer reads its snapshot directory (`LATEST_HEIGHT` file -> `snapshot_<height>/METADATA`)
   and responds with `MetadataResponse` containing filenames, MD5 checksums, and height
3. Syncing node stores the metadata, obtains light-block-verified state from a P2P state provider,
   and begins requesting individual files from the peer
4. Each file is verified against the MD5 checksum from metadata
5. Files are written to `<db-dir>/application.db/` and `<root>/wasm/wasm/state/wasm/` (wasm files
   have `_wasm` suffix)
6. When all files are synced, `postSyncFn` is called, which:
   - Bootstraps state store
   - Saves seen commit
   - Publishes state sync completion event
   - Calls `LoadLatest` ABCI method
   - Marks mempool reactor ready
   - Switches to block sync

### 4.3 Configuration

```toml
[db-sync]
db-sync-enable = false
snapshot-interval = 0            # Reserved for future
snapshot-directory = ""          # Path to snapshot files on serving node
snapshot-worker-count = 16       # Parallel workers for snapshot creation
timeout-in-seconds = 1200        # Metadata timeout
no-file-sleep-in-seconds = 1     # Sleep between file polls
file-worker-count = 32           # Parallel file download workers
file-worker-timeout = 30         # Per-file download timeout
trust-height = 0                 # Light client trust anchor
trust-hash = ""
trust-period = "86400s"
verify-light-block-timeout = "60s"
blacklist-ttl = "5m"
```

### 4.4 Snapshot CLI Command

`tendermint snapshot <height>` -- creates a DB sync snapshot at the given height by copying
application DB files and wasm state files into a snapshot directory with MD5 checksums.

**Important caveat (noted in code TODO):** The snapshot includes wasm files, meaning Tendermint
is aware of CosmWasm internals. The code acknowledges this is architecturally wrong and should
be handled at the Cosmos SDK layer via new ABCI methods.

---

## 5. Self-Remediation System

**Files:** `config/config.go`, `internal/blocksync/reactor.go`, `internal/p2p/pex/reactor.go`, `internal/statesync/reactor.go`

Sei adds automatic self-healing behavior across multiple reactors via a `SelfRemediationConfig`:

```go
type SelfRemediationConfig struct {
    P2pNoPeersRestarWindowSeconds        uint64  // PEX: restart if no peers for this long
    StatesyncNoPeersRestartWindowSeconds uint64  // Statesync: restart if no peers for this long
    BlocksBehindThreshold                uint64  // Blocksync: restart if this many blocks behind
    BlocksBehindCheckIntervalSeconds     uint64  // How often to check (default: 60s)
    RestartCooldownSeconds               uint64  // Min time between restarts (default: 600s)
}
```

**Restart mechanism:** Each reactor has a `restartCh chan struct{}` that signals to the node. The
node's `routerRestartCh` triggers a controlled restart of the P2P router and services.

### 5.1 Block Sync: Blocks-Behind Restart

If the node falls behind by more than `BlocksBehindThreshold` blocks (compared to max peer height),
it sends a restart signal. Checks happen every `BlocksBehindCheckIntervalSeconds`. A cooldown
(`RestartCooldownSeconds`) prevents restart storms.

### 5.2 PEX: No-Peers Restart

If the PEX reactor has zero available peers for longer than `P2pNoPeersRestarWindowSeconds`, it
signals a restart. This handles the case where the P2P layer becomes partitioned.

### 5.3 State Sync: No-Peers Restart

Same pattern as PEX: if state sync has no peers for longer than
`StatesyncNoPeersRestartWindowSeconds`, it restarts.

---

## 6. P2P Changes

### 6.1 Channel Registration via `AddChDescToBeAdded`

**File:** `internal/p2p/router.go`

The router gains a deferred channel registration mechanism:

```go
func (r *Router) AddChDescToBeAdded(chDesc *ChannelDescriptor, callback func(*Channel))
```

Instead of opening channels at construction time, channel descriptors and their setup callbacks
are collected and opened during `OnStart()`. This prevents race conditions where channels are used
before the router's transport is ready.

**FORKED_CHANGELOG note:** "Fix open connection race conditions within p2p channels by waiting
synchronously for descriptors to be registered before establishing peer connections"

### 6.2 DB Sync Channels

Four new P2P channel IDs are registered for DB sync: 0x70, 0x71, 0x72, 0x73.

### 6.3 Peer Gossip Sleep (Backport)

**FORKED_CHANGELOG:** "Backport add peer gossip sleep" from CometBFT PR #241. The
`PeerGossipSleepDuration` config controls sleep between gossip rounds in the consensus reactor.

---

## 7. Block/Data Structure Changes

### 7.1 Proposal Structure

The `Proposal` type is significantly extended:

```go
type Proposal struct {
    // Standard fields...
    TxKeys          []TxKey          // NEW: transaction hashes for key-only gossip
    Header          Header           // NEW: full header for block reconstruction
    LastCommit      *Commit          // NEW: last commit for block reconstruction
    Evidence        EvidenceList     // NEW: evidence list for block reconstruction
    ProposerAddress Address          // NEW: original proposer address
}
```

### 7.2 Data.Hash() Takes `overwrite` Parameter

**File:** `types/block.go`

```go
func (data *Data) Hash(overwrite bool) tmbytes.HexBytes
```

The `overwrite` parameter forces recomputation of the cached hash. Upstream has no parameter.

### 7.3 Block.GetTxKeys() Helper

**File:** `types/block.go`

```go
func (b *Block) GetTxKeys() []TxKey
```

Extracts SHA-256 keys from all transactions in the block.

---

## 8. State Execution Changes

### 8.1 Mempool-Backed Block Building Helpers

**File:** `internal/state/execution.go`

```go
func (blockExec *BlockExecutor) GetMissingTxs(txKeys []types.TxKey) []types.TxKey
func (blockExec *BlockExecutor) SafeGetTxsByKeys(txKeys []types.TxKey) (types.Txs, []types.TxKey)
func (blockExec *BlockExecutor) CheckTxFromPeerProposal(ctx context.Context, tx types.Tx)
```

- `GetMissingTxs`: Returns which TxKeys are NOT in the mempool
- `SafeGetTxsByKeys`: Fetches transactions from mempool by key, also returning missing keys
- `CheckTxFromPeerProposal`: Submits a transaction from a peer's proposal through CheckTx
  (ignoring errors from duplicate insertion)

---

## 9. Configuration Additions

### 9.1 New Top-Level Config Sections

- `[db-sync]` -- DB sync configuration (see section 4.3)
- `[self-remediation]` -- Self-healing configuration (see section 5)

### 9.2 New Consensus Config

- `gossip-tx-key-only` (bool, default true) -- Enable TxKey-based proposal gossip

### 9.3 New Mempool Config

- `tx-notify-threshold` (uint64, default 0) -- Min txs before reaping
- `check-tx-error-blacklist-enabled` (bool, default false)
- `check-tx-error-threshold` (int, default 0)
- `pending-size` (int, default 5000) -- Max pending transactions
- `max-pending-txs-bytes` (int64, default 1GB) -- Max pending tx bytes
- `pending-ttl-duration` (duration, default 0)
- `pending-ttl-num-blocks` (int64, default 0)
- `remove-expired-txs-from-queue` (bool, default true)
- `duplicate-txs-cache-size` (int, default 100000)

---

## 10. Dependency Changes

**File:** `go.mod`

Notable additions beyond upstream Tendermint v0.35:
- `github.com/grafana/pyroscope-go/godeltaprof v0.1.8` -- Continuous profiling support (imported in node.go as pprof handler)
- `github.com/patrickmn/go-cache v2.1.0` -- TTL cache for duplicate transaction detection
- `go.opentelemetry.io/otel v1.9.0` -- OpenTelemetry tracing
- `go.opentelemetry.io/otel/sdk v1.9.0`
- `go.opentelemetry.io/otel/trace v1.9.0`

---

## 11. New Packages/Files (Not in Upstream)

- `internal/dbsync/` -- Entire DB sync subsystem (reactor, syncer, snapshot)
- `proto/tendermint/dbsync/` -- DB sync protobuf definitions
- `utils/slice.go` -- Generic `Map` helper function
- `export/export.go` -- Re-exports internal types (`BlockStore`, `Store`, `Query`, `JsonMarshal`)
  for use by external packages (sei-chain)
- `cmd/tendermint/commands/snapshot.go` -- Snapshot CLI command

---

## 12. Hard Rollback Support

**FORKED_CHANGELOG:** "Hard rollback by deleting app and block states" -- Cherry-picked from
[tendermint/tendermint#9261](https://github.com/tendermint/tendermint/pull/9261) with modifications.
This allows recovering from a corrupted state by deleting block and app state.

---

## Summary: Impact on Node Controller

| Area | Relevance to Node Controller |
|------|------------------------------|
| `gossip-tx-key-only` | Must be configured correctly for validators; affects block propagation latency |
| DB sync | Alternative to state sync for bootstrapping; controller must set `db-sync-enable` and manage snapshot directory |
| `LoadLatest` ABCI | Called after DB sync; application must reload state from disk |
| Self-remediation | Controller should configure thresholds appropriately; restart signals may conflict with external restart orchestration |
| Pending txs / EVM nonce ordering | No direct controller config needed, but explains mempool behavior |
| Block part size (1MB) | Affects network bandwidth requirements |
| OpenTelemetry tracing | Can be configured via TracerProviderOptions when constructing the node |
| Pyroscope profiling | pprof endpoint includes godeltaprof handlers |
