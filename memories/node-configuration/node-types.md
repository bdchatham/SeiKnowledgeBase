---
topic: "Sei node types: modes, roles, and configuration differences"
sources:
  - repo: sei-tendermint
    version: v0.6.4
    files:
      - config/config.go
      - node/node.go
      - node/seed.go
      - node/setup.go
  - repo: sei-cosmos
    version: v0.3.66
    files:
      - server/config/config.go
      - store/types/pruning.go
  - repo: sei-config
    version: v0.0.9-0.20260327015454-7cf35ff77daa
    files:
      - config.go
      - types.go
      - defaults.go
      - enrichments.go
      - validate.go
      - intent.go
      - io.go
      - chains.go
      - registry.go
      - resolve.go
      - CLAUDE.md
  - repo: sei-chain
    version: v0.0.38
    files:
      - app/seidb.go
      - cmd/seid/cmd/root.go
verified: 2026-04-07
confidence: high
---

# Sei Node Types: Modes, Roles, and Configuration Differences

## 1. Enumeration of All Node Modes

Sei recognizes exactly **four** node operating modes, defined in sei-config `types.go`:

```go
const (
    ModeValidator NodeMode = "validator"
    ModeFull      NodeMode = "full"
    ModeSeed      NodeMode = "seed"
    ModeArchive   NodeMode = "archive"
)
```

At the sei-tendermint level, only three modes exist (archive is a Sei-layer concept):

```go
// sei-tendermint config/config.go
const (
    ModeFull      = "full"
    ModeValidator = "validator"
    ModeSeed      = "seed"
)
```

**Archive mode** is a sei-config abstraction that builds on top of `full` mode with specific storage overrides. It does not exist as a distinct mode in Tendermint proper.

There is a helper method `IsFullnodeType()` that groups `full` and `archive` together:

```go
func (m NodeMode) IsFullnodeType() bool {
    switch m {
    case ModeFull, ModeArchive:
        return true
    default:
        return false
    }
}
```

### Additional Sync Mechanisms (Not Modes)

Two additional sync mechanisms exist as **configuration flags**, not separate modes. A node in any mode can use these during bootstrap:

- **State Sync**: `statesync.enable = true` in config.toml (bootstraps from snapshots)
- **DB Sync**: `db-sync.db-sync-enable = true` in config.toml (imports DB files directly, Sei-specific)

These are mutually exclusive -- the node panics if both are enabled simultaneously:

```go
// sei-tendermint node/node.go
if stateSync && shoulddbsync {
    panic("statesync and dbsync cannot be turned on at the same time")
}
```

---

## 2. Node Type Detailed Breakdown

### 2.1 Validator Node (`mode: validator`)

**Purpose**: Participates in consensus by proposing and signing blocks.

**Tendermint-level behavior** (sei-tendermint `node/node.go`):
- Uses `makeNode()` (not `makeSeedNode()`)
- Loads or generates `priv_validator_key.json` and `priv_validator_state.json`
- Sets the private validator on the consensus state: `csState.SetPrivValidator(ctx, privValidator)`
- Starts all reactors: consensus, block sync, state sync, evidence, mempool, PEX (if enabled)
- Requires a valid public key; fails with error if `pubKey == nil`
- Can connect to an external signer via socket or gRPC (`priv-validator.laddr`)

**sei-config mode overrides** (`defaults.go:applyValidatorOverrides`):

```go
func applyValidatorOverrides(cfg *SeiConfig) {
    cfg.TxIndex.Indexer = []string{"null"}        // no tx indexing
    cfg.Network.RPC.ListenAddress = "tcp://0.0.0.0:26657"
    cfg.Network.P2P.ListenAddress = "tcp://0.0.0.0:26656"
    cfg.Network.P2P.AllowDuplicateIP = false       // strict IP dedup

    cfg.API.REST.Enable = false                    // no REST API
    cfg.API.GRPC.Enable = false                    // no gRPC
    cfg.API.GRPCWeb.Enable = false                 // no gRPC-Web
    cfg.Storage.StateStore.Enable = false           // no SS store (SeiDB)

    cfg.EVM.HTTPEnabled = false                    // no EVM RPC
    cfg.EVM.WSEnabled = false                      // no EVM WebSocket
}
```

**Key characteristics**:
- Tx indexing disabled (`"null"`) -- validators don't need to serve queries
- All query APIs disabled (REST, gRPC, gRPC-Web, EVM RPC)
- SeiDB state store disabled (only state-commit needed for consensus)
- AllowDuplicateIP = false (security: prevents multiple connections from same IP)
- Pruning strategy: inherited from base defaults (`nothing`) -- validators keep full state by default
- The validation layer warns if EVM RPC is enabled on a validator (see `validateEVM`)

**Validation warning**:
```go
if cfg.Mode == ModeValidator && (e.HTTPEnabled || e.WSEnabled) {
    r.addWarning("evm", "EVM RPC is enabled on a validator node; "+
        "this is unusual and may increase attack surface")
}
```

### 2.2 Full Node (`mode: full`)

**Purpose**: Participates in the network, serves queries, but does not sign blocks.

**Tendermint-level behavior**:
- Uses `makeNode()` (not `makeSeedNode()`)
- Does NOT load private validator keys (`makeDefaultPrivval` returns nil for non-validator modes)
- Starts all reactors (same as validator minus signing)
- PEX enabled by default for peer discovery

**sei-config mode overrides** (`defaults.go:applyFullOverrides`):

```go
func applyFullOverrides(cfg *SeiConfig) {
    cfg.TxIndex.Indexer = []string{"kv"}           // tx indexing enabled
    cfg.Network.RPC.ListenAddress = "tcp://0.0.0.0:26657"
    cfg.Network.P2P.ListenAddress = "tcp://0.0.0.0:26656"

    cfg.API.REST.Enable = true                     // REST API enabled
    cfg.API.GRPC.Enable = true                     // gRPC enabled
    cfg.API.GRPCWeb.Enable = true                  // gRPC-Web enabled
    cfg.Storage.StateStore.Enable = true            // SeiDB SS store enabled
    cfg.Storage.StateStore.KeepRecent = 100_000     // keep 100k recent versions
    cfg.Chain.MinRetainBlocks = 100_000             // keep 100k blocks

    cfg.EVM.HTTPEnabled = true                     // EVM JSON-RPC
    cfg.EVM.WSEnabled = true                       // EVM WebSocket
}
```

**Key characteristics**:
- All query APIs enabled (REST, gRPC, gRPC-Web, EVM HTTP, EVM WS)
- Tx indexing enabled with KV backend
- SeiDB state store enabled with 100k recent versions
- MinRetainBlocks = 100,000 (Tendermint-level block pruning)
- Base pruning strategy is `nothing` (from `baseDefaults`), but `KeepRecent=100_000` on the state store means historical versions beyond 100k are pruned in SeiDB SS

### 2.3 Archive Node (`mode: archive`)

**Purpose**: Full node that retains complete historical state for deep queries and tracing.

**sei-config mode overrides** (`defaults.go:applyArchiveOverrides`):

```go
func applyArchiveOverrides(cfg *SeiConfig) {
    applyFullOverrides(cfg)                        // start with full node settings

    cfg.Storage.PruningStrategy = PruningNothing    // keep all state
    cfg.Storage.StateStore.KeepRecent = 0           // 0 = keep all versions
    cfg.Chain.MinRetainBlocks = 0                   // 0 = keep all blocks
    cfg.EVM.MaxTraceLookbackBlocks = -1             // -1 = unlimited trace lookback
}
```

**Key characteristics**:
- Inherits everything from full node mode
- Pruning completely disabled (`nothing` strategy, KeepRecent=0, MinRetainBlocks=0)
- Unlimited EVM trace lookback (`MaxTraceLookbackBlocks = -1`), critical for `debug_traceBlockByNumber` and similar calls
- This is the only mode where `-1` for trace lookback is the default
- Disk usage grows without bound -- archive nodes need significantly more storage

**Important distinction**: Archive is NOT a separate Tendermint mode. At the sei-tendermint level, it runs as `mode: full`. The archive-specific behavior is entirely driven by app-level config (pruning, state store retention, EVM trace depth).

### 2.4 Seed Node (`mode: seed`)

**Purpose**: Dedicated peer discovery node. Crawls the network and shares peer addresses via the PEX (Peer Exchange) protocol. Does not participate in consensus or serve application queries.

**Tendermint-level behavior** (sei-tendermint `node/seed.go`):
- Uses a completely different struct: `seedNodeImpl` instead of `nodeImpl`
- Uses `makeSeedNode()` instead of `makeNode()`
- **Only starts P2P and PEX reactors** -- no consensus, block sync, state sync, evidence, mempool, or indexer
- PEX must be enabled; startup fails if disabled: `"cannot run seed nodes with PEX disabled"`
- Creates genesis state from doc but does NOT do handshake or replay
- NodeInfo advertises only `PexChannel` (not the full set of channels)
- Seed-specific NodeInfo creation via `makeSeedNodeInfo()` reports TxIndex as `"off"`

```go
// seed.go: seed node only has PEX channel
Channels: []byte{
    pex.PexChannel,
},
```

Compared to full/validator NodeInfo which has 11 channels:
```go
// setup.go: full/validator node channels
Channels: []byte{
    byte(blocksync.BlockSyncChannel),
    byte(consensus.StateChannel),
    byte(consensus.DataChannel),
    byte(consensus.VoteChannel),
    byte(consensus.VoteSetBitsChannel),
    byte(mempool.MempoolChannel),
    byte(evidence.EvidenceChannel),
    byte(statesync.SnapshotChannel),
    byte(statesync.ChunkChannel),
    byte(statesync.LightBlockChannel),
    byte(statesync.ParamsChannel),
},
```

**sei-config mode overrides** (`defaults.go:applySeedOverrides`):

```go
func applySeedOverrides(cfg *SeiConfig) {
    cfg.TxIndex.Indexer = []string{"null"}         // no tx indexing
    cfg.Network.P2P.MaxConnections = 1000          // high connection limit
    cfg.Network.P2P.AllowDuplicateIP = true        // allow multiple peers per IP

    cfg.API.REST.Enable = false                    // no REST API
    cfg.API.GRPC.Enable = false                    // no gRPC
    cfg.API.GRPCWeb.Enable = false                 // no gRPC-Web
    cfg.Storage.StateStore.Enable = false           // no SeiDB SS store
    cfg.Storage.PruningStrategy = PruningEverything // aggressive pruning

    cfg.EVM.HTTPEnabled = false                    // no EVM RPC
    cfg.EVM.WSEnabled = false                      // no EVM WebSocket
}
```

**Key characteristics**:
- MaxConnections bumped to 1000 (vs 100 default) -- seed nodes need many connections
- AllowDuplicateIP = true (seed nodes often run in environments with NAT)
- Pruning strategy = `everything` (most aggressive -- seeds don't need state)
- All query APIs and EVM RPC disabled
- No state store, no tx indexing
- Lightest resource footprint of any node type

---

## 3. Configuration Layers and Their Relationship

Sei node configuration spans three layers, which sei-config unifies:

### 3.1 config.toml (Tendermint layer, sei-tendermint)

Managed by the `Config` struct in sei-tendermint `config/config.go`. Contains:

| Section | Key Fields |
|---------|-----------|
| BaseConfig | `mode`, `moniker`, `proxy-app`, `db-backend`, `log-level`, `log-format`, `filter-peers` |
| `[rpc]` | `laddr`, `cors-allowed-origins`, `unsafe`, `max-open-connections`, `event-log-window-size`, `lag-threshold` |
| `[p2p]` | `laddr`, `external-address`, `persistent-peers`, `bootstrap-peers`, `blocksync-peers`, `max-connections`, `pex`, `private-peer-ids`, `allow-duplicate-ip`, `unconditional-peer-ids`, `send-rate`, `recv-rate` |
| `[mempool]` | `broadcast`, `size`, `max-txs-bytes`, `max-tx-bytes`, `ttl-duration`, `ttl-num-blocks`, `pending-size`, `check-tx-error-blacklist-enabled` |
| `[statesync]` | `enable`, `use-p2p`, `rpc-servers`, `trust-height`, `trust-hash`, `trust-period`, `backfill-blocks`, `use-local-snapshot` |
| `[consensus]` | `create-empty-blocks`, `gossip-tx-key-only`, `double-sign-check-height`, `unsafe-*-timeout-override` |
| `[tx-index]` | `indexer` (null, kv, psql) |
| `[instrumentation]` | `prometheus`, `prometheus-listen-addr`, `namespace` |
| `[priv-validator]` | `key-file`, `state-file`, `laddr` (for remote signer) |
| `[self-remediation]` | `p2p-no-peers-available-window-seconds`, `blocks-behind-threshold`, `restart-cooldown-seconds` |
| `[db-sync]` | `db-sync-enable`, `trust-height`, `trust-hash` (Sei-specific) |

### 3.2 app.toml (Cosmos SDK + Sei layer, sei-cosmos)

Managed by the `Config` struct in sei-cosmos `server/config/config.go`. Contains:

| Section | Key Fields |
|---------|-----------|
| BaseConfig | `minimum-gas-prices`, `pruning`, `pruning-keep-recent`, `pruning-keep-every`, `pruning-interval`, `halt-height`, `halt-time`, `min-retain-blocks`, `inter-block-cache`, `iavl-cache-size`, `concurrency-workers`, `occ-enabled` |
| `[api]` | `enable` (REST), `swagger`, `address`, `enabled-unsafe-cors` |
| `[grpc]` | `enable`, `address` |
| `[grpc-web]` | `enable`, `address` |
| `[state-sync]` | `snapshot-interval`, `snapshot-keep-recent`, `snapshot-directory` |
| `[state-commit]` | `enable` (SeiDB SC), `directory`, `zero-copy`, `snapshot-keep-recent`, `snapshot-interval` |
| `[state-store]` | `enable` (SeiDB SS), `db-directory`, `backend`, `keep-recent`, `prune-interval-seconds` |
| `[telemetry]` | `enabled`, `prometheus-retention-time`, `service-name` |
| `[genesis]` | `stream-import`, `genesis-stream-file` |

### 3.3 Unified sei.toml (sei-config, future)

sei-config's `SeiConfig` struct merges both layers into a single type with a unified key namespace. Currently, `ReadConfigFromDir()` / `WriteConfigToDir()` still read/write the two-file layout (config.toml + app.toml) via legacy types in `legacy.go`. The mapping is:

- `config.toml` fields -> `SeiConfig.Network`, `SeiConfig.Consensus`, `SeiConfig.Mempool`, `SeiConfig.StateSync` (Tendermint statesync), `SeiConfig.TxIndex`, `SeiConfig.Metrics`, `SeiConfig.PrivValidator`, `SeiConfig.SelfRemediation`
- `app.toml` fields -> `SeiConfig.Chain` (gas prices, pruning), `SeiConfig.Storage`, `SeiConfig.API`, `SeiConfig.EVM`, `SeiConfig.WASM`, `SeiConfig.GigaExecutor`, `SeiConfig.LightInvariance`, `SeiConfig.Genesis`

### 3.4 ConfigIntent Pipeline

The controller and sidecar use sei-config's intent resolution pipeline:

```
ConfigIntent{Mode, Overrides} --> ResolveIntent() --> ConfigResult{SeiConfig}
```

1. `DefaultForMode(mode)` generates baseline config
2. `ApplyOverrides(cfg, overrides)` patches it with flat key-value overrides
3. `ValidateWithOpts(cfg)` produces diagnostics
4. Result is written to disk via `WriteConfigToDir()` (produces config.toml + app.toml)

---

## 4. Pruning Strategies

Pruning is a critical differentiator between node types. There are two independent pruning systems:

### 4.1 Cosmos SDK Pruning (app.toml `pruning`)

Defined in sei-cosmos `store/types/pruning.go`:

| Strategy | KeepRecent | KeepEvery | Interval | Effect |
|----------|-----------|-----------|----------|--------|
| `default` | 362880 | 0 | 10 | Keep ~25 days of state (at 6s blocks), prune every 10 blocks |
| `everything` | 2 | 0 | 10 | Keep only 2 recent versions, prune aggressively |
| `nothing` | 0 | 1 | 0 | Keep all versions, never prune |
| `custom` | user-defined | user-defined | user-defined | Fully customizable |

### 4.2 SeiDB State Store Pruning (app.toml `state-store`)

When SeiDB is enabled (`state-commit.enable = true` and `state-store.enable = true`):

- `state-store.keep-recent`: Number of recent versions to retain (0 = all)
- `state-store.prune-interval-seconds`: How often to run pruning (default: 600)

### 4.3 Tendermint Block Pruning (config.toml via `min-retain-blocks`)

- `min-retain-blocks` in app.toml controls how many blocks Tendermint keeps
- 0 = keep all blocks
- Set by sei-config: 100,000 for full nodes, 0 for archive/validator

### Mode defaults for pruning:

| Mode | `storage.pruning` | `storage.state_store.keep_recent` | `chain.min_retain_blocks` |
|------|-------------------|----------------------------------|--------------------------|
| validator | `nothing` | N/A (SS disabled) | 0 (keep all) |
| full | `nothing` | 100,000 | 100,000 |
| archive | `nothing` | 0 (keep all) | 0 (keep all) |
| seed | `everything` | N/A (SS disabled) | 0 (default) |

---

## 5. Parameter Interaction Matrix

### 5.1 State Sync Parameters

When `state_sync.enable = true`:
- Requires `trust_height > 0`, `trust_hash` (hex), `trust_period > 0`
- If not using P2P (`use_p2p = false`): requires at least 2 `rpc_servers`
- Cannot be combined with `db-sync.enable = true` (panic)
- State sync is skipped if `state.LastBlockHeight > 0` (node already has state)
- State sync is skipped if the node is the only validator

### 5.2 Snapshot Generation vs Consumption

Snapshots have two roles -- generation (serving) and consumption (bootstrapping):

**Generation** (a running node creating snapshots for others):
- Controlled by `storage.snapshot_interval` (blocks between snapshots) and `storage.snapshot_keep_recent`
- Cannot use `pruning = everything` with `snapshot_interval > 0` (validation error)
- When SeiDB SC is enabled and snapshot interval > 0, SeiDB SS must also be enabled (enforced in sei-chain `app/seidb.go`)
- sei-config provides `SnapshotGenerationOverrides()` helper:
  ```go
  func SnapshotGenerationOverrides(keepRecent int32) map[string]string {
      return map[string]string{
          "storage.pruning":              PruningNothing,
          "storage.snapshot_interval":    strconv.FormatInt(DefaultSnapshotInterval, 10), // 2000
          "storage.snapshot_keep_recent": strconv.FormatInt(int64(keepRecent), 10),
      }
  }
  ```

**Consumption** (bootstrapping from a snapshot):
- Controlled by `state_sync.enable = true` + trust parameters
- `state_sync.use_local_snapshot = true` uses an existing local snapshot instead of discovering from peers

### 5.3 PEX and Node Identity

| Mode | PEX Required | PEX Default | AllowDuplicateIP |
|------|-------------|-------------|------------------|
| validator | No | true (base) | false |
| full | No | true (base) | false (base) |
| archive | No | true (base) | false (base) |
| seed | **Yes** (hard requirement) | true | true |

If `pex = false` and mode is seed, the node fails to start:
```go
if !cfg.P2P.PexReactor {
    return nil, errors.New("cannot run seed nodes with PEX disabled")
}
```

### 5.4 Tx Indexer Settings

| Mode | Indexer | Effect |
|------|---------|--------|
| validator | `["null"]` | No indexing, cannot query historical txs |
| full | `["kv"]` | KV-backed indexing, supports tx queries |
| archive | `["kv"]` | KV-backed indexing (inherits from full) |
| seed | `["null"]` | No indexing (seed doesn't process txs) |

The indexer also supports `"psql"` for PostgreSQL-backed indexing, configurable via `tx_index.psql_conn`.

### 5.5 API Enablement Matrix

| Mode | REST API | gRPC | gRPC-Web | EVM HTTP | EVM WS | RPC (Tendermint) |
|------|----------|------|----------|----------|--------|-----------------|
| validator | off | off | off | off | off | on (0.0.0.0) |
| full | on | on | on | on | on | on (0.0.0.0) |
| archive | on | on | on | on | on | on (0.0.0.0) |
| seed | off | off | off | off | off | on (base default) |

Note: The Tendermint RPC (`network.rpc.listen_address`) is always available (base default `tcp://127.0.0.1:26657`). The validator/full/archive modes override it to `0.0.0.0:26657` (externally accessible). The seed mode does not override it.

---

## 6. Sei-Specific Features Not in Upstream Cosmos/Tendermint

### 6.1 DB Sync (sei-tendermint)

A Sei-specific alternative to state sync. Imports raw database files from peers:

```go
type DBSyncConfig struct {
    Enable              bool          `mapstructure:"db-sync-enable"`
    SnapshotInterval    int           `mapstructure:"snapshot-interval"`
    SnapshotDirectory   string        `mapstructure:"snapshot-directory"`
    SnapshotWorkerCount int           `mapstructure:"snapshot-worker-count"`
    TimeoutInSeconds    int           `mapstructure:"timeout-in-seconds"`
    FileWorkerCount     int           `mapstructure:"file-worker-count"`
    TrustHeight         int64         `mapstructure:"trust-height"`
    TrustHash           string        `mapstructure:"trust-hash"`
    TrustPeriod         time.Duration `mapstructure:"trust-period"`
}
```

DB sync uses its own light block verification (separate from state sync), has dedicated P2P channels for metadata and file transfer, and calls `LoadLatest` on the ABCI app after import.

### 6.2 SeiDB (State Commit + State Store)

Sei replaces IAVL with MemIAVL (state-commit layer) and adds a separate historical state store:

- **State Commit** (`storage.state_commit`): Replaces IAVL. Uses MemIAVL for the working state.
  - `enable`: Activates SeiDB (replaces standard Cosmos store)
  - `write_mode` / `read_mode`: Controls EVM data routing (`cosmos_only`, `dual_write`, `split_write`, `evm_only`)

- **State Store** (`storage.state_store`): Historical state backend for queries.
  - `enable`: Required for query nodes; disabled for validators and seeds
  - `backend`: `pebbledb` (default) or `rocksdb`
  - `keep_recent`: How many versions to keep (0 = all)
  - `prune_interval_seconds`: Pruning frequency

### 6.3 EVM RPC (sei-chain)

Sei includes a full EVM JSON-RPC server alongside the Cosmos RPC:

- HTTP server on port 8545 (configurable via `evm.http_port`)
- WebSocket server on port 8546 (configurable via `evm.ws_port`)
- Supports tracing (`debug_traceBlockByNumber`, etc.) with configurable `max_trace_lookback_blocks`
- `max_trace_lookback_blocks = -1` (archive mode) enables unlimited historical tracing
- `deny_list` allows blocking specific RPC methods at runtime (hot-reloadable)

### 6.4 Gossip Transaction Key Only (sei-tendermint)

Sei-specific consensus optimization:

```go
GossipTransactionKeyOnly bool `mapstructure:"gossip-tx-key-only"`
```

Default: `true` in Sei (vs standard Tendermint which gossips full transactions). Reduces bandwidth by only gossiping transaction hashes.

### 6.5 Self-Remediation (sei-tendermint)

Automated restart behaviors unique to Sei:

- `p2p-no-peers-available-window-seconds`: Restart if no P2P peers for N seconds
- `statesync-no-peers-available-window-seconds`: Restart if no statesync peers for N seconds
- `blocks-behind-threshold`: Restart if node falls N blocks behind
- `blocks-behind-check-interval-seconds`: How often to check (default: 60)
- `restart-cooldown-seconds`: Minimum time between restarts (default: 600)

### 6.6 GigaExecutor

Sei-specific parallel execution engine:

```go
type GigaExecutorConfig struct {
    Enabled    bool `toml:"enabled"`
    OccEnabled bool `toml:"occ_enabled"`
}
```

### 6.7 Light Invariance Checking

```go
type LightInvarianceConfig struct {
    SupplyEnabled bool `toml:"supply_enabled"`
}
```

Default: enabled. Performs lightweight supply invariance checks.

### 6.8 Block Sync Peers (sei-tendermint)

Sei adds a dedicated peer list for block sync (not in upstream Tendermint):

```go
BlockSyncPeers string `mapstructure:"blocksync-peers"`
```

Separate from `persistent-peers` and `bootstrap-peers`, these peers are used specifically for block synchronization.

---

## 7. Well-Known Chains

sei-config embeds genesis data for known chains:

| Chain ID | Network | RPC |
|----------|---------|-----|
| `pacific-1` | Mainnet | `https://rpc.sei-apis.com` |
| `atlantic-2` | Testnet | `https://rpc-testnet.sei-apis.com` |
| `arctic-1` | Devnet | `https://rpc-arctic-1.sei-apis.com` |

Genesis is embedded via `//go:embed chains/*/genesis.json` and accessed through `GenesisForChain(chainID)`. Unknown chains require a custom genesis source.

---

## 8. Configuration Defaults Summary Table

All values from `baseDefaults()` in sei-config `defaults.go`, with mode overrides noted:

| Field | Base Default | Validator | Full | Archive | Seed |
|-------|-------------|-----------|------|---------|------|
| `storage.pruning` | `nothing` | nothing | nothing | nothing | **everything** |
| `storage.state_store.enable` | true | **false** | true | true | **false** |
| `storage.state_store.keep_recent` | 100,000 | N/A | 100,000 | **0** | N/A |
| `chain.min_retain_blocks` | 0 | 0 | **100,000** | **0** | 0 |
| `tx_index.indexer` | `["kv"]` | **["null"]** | ["kv"] | ["kv"] | **["null"]** |
| `api.rest.enable` | false | false | **true** | **true** | false |
| `api.grpc.enable` | true | **false** | **true** | **true** | **false** |
| `api.grpc_web.enable` | true | **false** | **true** | **true** | **false** |
| `evm.http_enabled` | true | **false** | true | true | **false** |
| `evm.ws_enabled` | true | **false** | true | true | **false** |
| `evm.max_trace_lookback_blocks` | 10,000 | 10,000 | 10,000 | **-1** | 10,000 |
| `network.p2p.max_connections` | 100 | 100 | 100 | 100 | **1000** |
| `network.p2p.allow_duplicate_ip` | false | **false** | false | false | **true** |
| `network.rpc.listen_address` | tcp://127.0.0.1:26657 | **0.0.0.0** | **0.0.0.0** | **0.0.0.0** | 127.0.0.1 |
| `network.p2p.listen_address` | tcp://127.0.0.1:26656 | **0.0.0.0** | **0.0.0.0** | **0.0.0.0** | 127.0.0.1 |
| `storage.state_commit.enable` | true | true | true | true | true |
| `storage.snapshot_interval` | 0 | 0 | 0 | 0 | 0 |
| `network.p2p.pex` | true | true | true | true | true |
| `consensus.gossip_transaction_key_only` | true | true | true | true | true |
| `chain.min_gas_prices` | 0.01usei | 0.01usei | 0.01usei | 0.01usei | 0.01usei |
| `chain.occ_enabled` | true | true | true | true | true |
| `metrics.enabled` | true | true | true | true | true |

---

## 9. Environment Variable Resolution

All config fields can be overridden via environment variables with the `SEI_` prefix (legacy `SEID_` also supported with deprecation warning). The naming convention maps from TOML key paths:

```
storage.state_store.keep_recent -> SEI_STORAGE_STATE_STORE_KEEP_RECENT
evm.http_port                   -> SEI_EVM_HTTP_PORT
chain.min_gas_prices            -> SEI_CHAIN_MIN_GAS_PRICES
```

This is handled by `ResolveEnv()` in sei-config `resolve.go`.

---

## 10. Sentry Node Pattern (Operational, Not a Mode)

There is no dedicated "sentry" mode in Sei. The sentry node pattern is achieved operationally by running a full node with specific P2P configuration:

- `persistent-peers`: Set to the validator's node ID
- `private-peer-ids`: Set to the validator's node ID (prevents gossiping validator's address)
- `pex = true`: Enable peer discovery for the sentry
- The validator sets `persistent-peers` to its sentries and optionally `pex = false`

This is a network topology pattern, not a config mode.
