---
topic: "sei-config package: configuration library internals"
sources:
  - repo: sei-config
    version: v0.0.9-0.20260327015454-7cf35ff77daa
    files:
      - config.go
      - types.go
      - defaults.go
      - validate.go
      - intent.go
      - registry.go
      - enrichments.go
      - migrate.go
      - resolve.go
      - io.go
      - legacy.go
      - chains.go
      - chains_test.go
      - config_test.go
      - intent_test.go
      - registry_test.go
      - migrate_test.go
      - go.mod
      - version.json
      - CLAUDE.md
verified: 2026-04-07
confidence: high
---

# sei-config Package Internals

## Overview

`sei-config` (`github.com/sei-protocol/sei-config`) is a shared Go library providing unified configuration types, mode-aware defaults, validation, and serialization for all Sei node components. It is consumed by three systems: `seid` (the node binary), `seictl` (CLI and sidecar), and `sei-k8s-controller` (Kubernetes operator).

**Single external dependency:** `github.com/BurntSushi/toml v1.5.0`

**Go version:** 1.25.6

**Config schema version:** `CurrentVersion = 1`

---

## 1. Top-Level Configuration Struct

`SeiConfig` is the unified configuration struct that merges all settings previously split across `config.toml` (Tendermint) and `app.toml` (Cosmos SDK + Sei extensions):

```go
type SeiConfig struct {
    Version int      `toml:"version"`
    Mode    NodeMode `toml:"mode"`

    Chain           ChainConfig           `toml:"chain"`
    Network         NetworkConfig         `toml:"network"`
    Consensus       ConsensusConfig       `toml:"consensus"`
    Mempool         MempoolConfig         `toml:"mempool"`
    StateSync       StateSyncConfig       `toml:"state_sync"`
    Storage         StorageConfig         `toml:"storage"`
    TxIndex         TxIndexConfig         `toml:"tx_index"`
    EVM             EVMConfig             `toml:"evm"`
    API             APIConfig             `toml:"api"`
    Metrics         MetricsConfig         `toml:"metrics"`
    Logging         LogConfig             `toml:"logging"`
    WASM            WASMConfig            `toml:"wasm"`
    GigaExecutor    GigaExecutorConfig    `toml:"giga_executor"`
    LightInvariance LightInvarianceConfig `toml:"light_invariance"`
    PrivValidator   PrivValidatorConfig   `toml:"priv_validator"`
    SelfRemediation SelfRemediationConfig `toml:"self_remediation"`
    Genesis         GenesisConfig         `toml:"genesis"`
}
```

The struct has 18 top-level sections plus `Version` and `Mode`.

---

## 2. Custom Types

### NodeMode

```go
type NodeMode string

const (
    ModeValidator NodeMode = "validator"
    ModeFull      NodeMode = "full"
    ModeSeed      NodeMode = "seed"
    ModeArchive   NodeMode = "archive"
)
```

Methods: `IsValid() bool`, `IsFullnodeType() bool` (returns true for `full` and `archive`), `String() string`.

### Duration

Wraps `time.Duration` for human-readable TOML serialization (e.g. `"10s"`, `"100ms"`, `"168h"`). Implements `MarshalText`/`UnmarshalText`. Helper `Dur(d time.Duration) Duration` constructs values.

### WriteMode / ReadMode

Control EVM data routing between Cosmos and EVM backends:

```go
// WriteMode: cosmos_only, dual_write, split_write, evm_only
// ReadMode: cosmos_only, evm_first, split_read
```

---

## 3. Configuration Loading

### Three Loading Pathways

1. **Mode defaults** (`DefaultForMode(mode)`) -- generates a complete config from scratch based on node mode
2. **Legacy file I/O** (`ReadConfigFromDir(homeDir)`) -- reads existing `config.toml` + `app.toml` from `homeDir/config/`
3. **Environment variable overlay** (`ResolveEnv(cfg)`) -- applies `SEI_*` env var overrides to an existing config

### Intent Resolution Pipeline (Primary Path)

The canonical way to produce configuration is through the **ConfigIntent** pipeline:

```go
type ConfigIntent struct {
    Mode          NodeMode          `json:"mode"`
    Overrides     map[string]string `json:"overrides,omitempty"`
    TargetVersion int               `json:"targetVersion,omitempty"`
    Incremental   bool              `json:"incremental,omitempty"`
}
```

Three resolution functions:

1. **`ResolveIntent(intent)`** -- Full bootstrap: mode defaults -> apply overrides -> validate -> return `ConfigResult`
2. **`ResolveIncrementalIntent(intent, current)`** -- Day-2 patches: takes existing config, applies overrides (does NOT regenerate from mode defaults)
3. **`ValidateIntent(intent)`** -- Dry-run validation: checks intent well-formedness without producing a resolved config

The pipeline:
1. Resolve target version (default: `CurrentVersion`)
2. Generate mode defaults via `DefaultForMode(mode)`
3. Set `cfg.Version = targetVersion`
4. Apply overrides via `ApplyOverrides(cfg, overrides)`
5. Validate via `ValidateWithOpts(cfg, opts)`
6. Return `ConfigResult{Config, Version, Mode, Diagnostics, Valid}`

**Important design rule:** The controller never calls `DefaultForMode`, `ApplyOverrides`, or `Validate` directly. It constructs a `ConfigIntent` and the sidecar resolves it.

### ApplyOverrides

```go
func ApplyOverrides(cfg *SeiConfig, overrides map[string]string) error
```

Takes a flat map of dotted TOML key paths to string values (e.g., `"evm.http_port" -> "9545"`). Uses the `Registry` to resolve each key to its Go struct field path, then sets it via reflection. Supports: string, bool, int/int64, uint/uint16/uint32/uint64, float64, Duration. Returns error for unknown keys or type parse failures.

### Environment Variable Resolution

```go
func ResolveEnv(cfg *SeiConfig) []string
```

Naming convention: `SEI_<SECTION>_<FIELD>` (e.g., `SEI_CHAIN_MIN_GAS_PRICES`). Also recognizes legacy `SEID_*` prefix with lower precedence and emits deprecation warnings. `SEI_` takes priority over `SEID_` when both are set.

### Legacy File I/O

```go
func ReadConfigFromDir(homeDir string) (*SeiConfig, error)
func WriteConfigToDir(cfg *SeiConfig, homeDir string) error
```

- Reads `homeDir/config/config.toml` (Tendermint) and `homeDir/config/app.toml` (Cosmos SDK) via intermediate `legacyTendermintConfig` and `legacyAppConfig` types
- Writes are atomic (temp file + `fsync` + rename) to prevent corruption
- Legacy TOML tags use **hyphens** (e.g., `persistent-peers`, `trust-height`), matching what existing `seid` binaries expect
- **Archive mode handling:** Tendermint only understands `validator/full/seed`. Archive mode is stored in `[sei]` section of `app.toml` as metadata and maps to `"full"` in `config.toml`'s mode field. On read, `[sei].mode` takes precedence if valid.

**Phase roadmap:**
- Phase 2 (current): Two-file layout (`config.toml` + `app.toml`)
- Phase 3 (future): Unified `sei.toml` format (IO layer switches internally, callers unchanged)

---

## 4. Mode-Aware Defaults

`DefaultForMode(mode)` calls `baseDefaults()` then `applyModeOverrides(cfg, mode)`.

### Base Defaults (shared by all modes)

| Section | Key fields | Default |
|---------|-----------|---------|
| **Chain** | `moniker` | hostname |
| | `proxy_app` | `tcp://127.0.0.1:26658` |
| | `abci` | `socket` |
| | `min_gas_prices` | `0.01usei` |
| | `inter_block_cache` | `true` |
| | `concurrency_workers` | `max(10, min(NumCPU*2, 128))` |
| | `occ_enabled` | `true` |
| **Network.RPC** | `listen_address` | `tcp://127.0.0.1:26657` |
| | `max_open_connections` | 900 |
| | `max_subscription_clients` | 100 |
| | `max_subscriptions_per_client` | 5 |
| | `timeout_broadcast_tx_commit` | 10s |
| | `max_body_bytes` | 1,000,000 |
| | `lag_threshold` | 300 blocks |
| **Network.P2P** | `listen_address` | `tcp://127.0.0.1:26656` |
| | `max_connections` | 100 |
| | `pex` | `true` |
| | `send_rate` / `recv_rate` | 20,971,520 bytes/sec |
| | `handshake_timeout` | 10s |
| | `dial_timeout` | 3s |
| | `queue_type` | `simple-priority` |
| **Consensus** | `wal_path` | `data/cs.wal/wal` |
| | `create_empty_blocks` | `true` |
| | `gossip_transaction_key_only` | `true` |
| | `peer_gossip_sleep_duration` | 100ms |
| **Mempool** | `size` | 5000 txs |
| | `max_txs_bytes` | 1 GiB |
| | `cache_size` | 10,000 |
| | `max_tx_bytes` | 1 MiB |
| | `ttl_duration` | 5s |
| | `ttl_num_blocks` | 10 |
| | `check_tx_error_blacklist_enabled` | `true` |
| **StateSync** | `trust_period` | 168h (7 days) |
| | `fetchers` | 2 |
| **Storage** | `db_backend` | `goleveldb` |
| | `pruning` | `nothing` |
| | `iavl_disable_fast_node` | `true` |
| | `state_commit.enable` | `true` |
| | `state_commit.write_mode` | `cosmos_only` |
| | `state_store.enable` | `true` |
| | `state_store.backend` | `pebbledb` |
| | `state_store.keep_recent` | 100,000 |
| | `state_store.prune_interval_seconds` | 600 |
| **TxIndex** | `indexer` | `["kv"]` |
| **EVM** | `http_enabled` | `true` |
| | `http_port` | 8545 |
| | `ws_enabled` | `true` |
| | `ws_port` | 8546 |
| | `simulation_gas_limit` | 10,000,000 |
| | `max_tx_pool_txs` | 1000 |
| | `max_blocks_for_log` | 2000 |
| | `max_trace_lookback_blocks` | 10,000 |
| | `worker_pool_size` | `min(64, NumCPU*2)` |
| **API** | `rest.enable` | `false` (base) |
| | `grpc.enable` | `true` |
| | `grpc.address` | `0.0.0.0:9090` |
| | `grpc_web.enable` | `true` |
| **Metrics** | `enabled` | `true` |
| | `prometheus_listen_addr` | `:26660` |
| | `namespace` | `tendermint` |
| **Logging** | `level` | `info` |
| | `format` | `plain` |
| **WASM** | `query_gas_limit` | 300,000 |
| **LightInvariance** | `supply_enabled` | `true` |
| **SelfRemediation** | `blocks_behind_check_interval_seconds` | 60 |
| | `restart_cooldown_seconds` | 600 |

### Mode Overrides

#### Validator
- TxIndex: `["null"]` (no indexing)
- RPC/P2P listen on `0.0.0.0`
- `allow_duplicate_ip`: false
- REST API, gRPC, gRPC-Web: **disabled**
- State Store: **disabled**
- EVM HTTP/WS: **disabled**

#### Seed
- TxIndex: `["null"]`
- `max_connections`: 1000
- `allow_duplicate_ip`: true
- REST API, gRPC, gRPC-Web: **disabled**
- State Store: **disabled**
- Pruning: `everything`
- EVM HTTP/WS: **disabled**

#### Full
- TxIndex: `["kv"]`
- RPC/P2P listen on `0.0.0.0`
- REST API, gRPC, gRPC-Web: **enabled**
- State Store: **enabled**, keep_recent=100,000
- `min_retain_blocks`: 100,000
- EVM HTTP/WS: **enabled**

#### Archive
- Same as Full, plus:
- Pruning: `nothing`
- `state_store.keep_recent`: 0 (keep all)
- `min_retain_blocks`: 0 (keep all)
- `evm.max_trace_lookback_blocks`: -1 (unlimited)

### Snapshot Generation Overrides

```go
func SnapshotGenerationOverrides(keepRecent int32) map[string]string
```

Returns overrides for nodes producing state-sync snapshots:
- `storage.pruning` = `nothing`
- `storage.snapshot_interval` = 2000 (blocks)
- `storage.snapshot_keep_recent` = `keepRecent`

---

## 5. Well-Known Ports

```go
const (
    PortEVMHTTP int32 = 8545
    PortEVMWS   int32 = 8546
    PortGRPC    int32 = 9090
    PortP2P     int32 = 26656
    PortRPC     int32 = 26657
    PortMetrics int32 = 26660
    PortSidecar int32 = 7777
)
```

`NodePorts()` returns the canonical set of 6 ports exposed by seid (excluding sidecar).

---

## 6. Validation

```go
func Validate(cfg *SeiConfig) *ValidationResult
func ValidateWithOpts(cfg *SeiConfig, opts ValidateOpts) *ValidationResult
```

Returns `*ValidationResult` with typed `Diagnostic` entries (never bare `error`).

### Severity Levels
- `SeverityError` -- prevents seid from starting
- `SeverityWarning` -- logged but does not block
- `SeverityInfo` -- informational

### Validation Rules

**Mode:** Must be one of: validator, full, seed, archive.

**Version:** Must be >= 1 and <= CurrentVersion (or opts.MaxVersion).

**Chain:**
- `min_gas_prices` must be non-empty
- `concurrency_workers` must be >= -1

**Network:**
- RPC: `max_open_connections`, `max_subscription_clients`, `max_subscriptions_per_client`, `timeout_broadcast_tx_commit`, `max_body_bytes`, `max_header_bytes` must be >= 0
- P2P: `flush_throttle_timeout`, `max_packet_msg_payload_size`, `send_rate`, `recv_rate` must be >= 0

**Consensus:**
- `unsafe_propose_timeout_override`, `unsafe_commit_timeout_override`, `create_empty_blocks_interval`, `peer_gossip_sleep_duration` must be >= 0
- `double_sign_check_height` must be >= 0

**Mempool:**
- `size`, `max_txs_bytes`, `cache_size`, `max_tx_bytes` must be >= 0
- `ttl_duration`, `ttl_num_blocks` must be >= 0
- `drop_utilisation_threshold` must be between 0.0 and 1.0

**StateSync (when enabled):**
- At least 2 RPC servers required when not using P2P
- `trust_period` must be > 0
- `trust_height` must be > 0
- `trust_hash` must be set and valid hex
- `fetchers` must be > 0
- `backfill_blocks` must be >= 0

**Storage:**
- Pruning strategy must be: default, nothing, everything, custom
- `state_commit.write_mode` and `read_mode` validated against enum values
- `state_store.write_mode` and `read_mode` validated against enum values
- `state_store.backend` warns if not pebbledb or rocksdb

**EVM:**
- `http_port` must be > 0 when HTTP enabled
- `ws_port` must be > 0 when WS enabled
- Warning if EVM RPC enabled on validator

**Metrics:** `max_open_connections` must be >= 0

**Logging:** `format` must be: plain, text, or json

**Cross-field:** Cannot enable snapshots with `everything` pruning strategy.

---

## 7. Field Registry

```go
func BuildRegistry() *Registry
```

Reflection-based discovery of all config fields from `SeiConfig` struct tags. Each field gets:

```go
type ConfigField struct {
    Key              string      // Dotted TOML key: "evm.http_port"
    EnvVar           string      // Env var name: "SEI_EVM_HTTP_PORT"
    FieldPath        string      // Go path: "EVM.HTTPPort"
    Type             FieldType   // string, int, uint, float, bool, duration, []string, other
    Description      string      // Human-readable (from enrichments)
    Unit             string      // "bytes", "blocks", "connections", etc.
    HotReload        bool        // Can change at runtime
    Deprecated       bool        // Scheduled for removal
    Section          string      // Top-level section: "evm", "storage", etc.
    SinceVersion     int         // Config version where field was introduced
    RequiredForModes []NodeMode  // Modes where field must be non-zero
}
```

### Query Methods
- `Fields()` -- all fields sorted by key
- `Field(key)` -- lookup by TOML key
- `FieldByEnvVar(envVar)` -- lookup by env var name
- `FieldsInSection(section)` -- all fields in a section
- `HotReloadableFields()` -- fields safe to change at runtime
- `DeprecatedFields()` -- fields marked deprecated
- `Sections()` -- unique section names
- `DefaultsByMode(mode)` -- default values as `map[string]any`

### Enrichment

`DefaultEnrichments()` returns curated metadata for key fields. After `BuildRegistry()`, call `registry.EnrichAll(DefaultEnrichments())` to populate descriptions, units, hot-reload flags, etc.

**Hot-reloadable fields** (from enrichments):
- `network.rpc.listen_address`
- `network.rpc.cors_allowed_origins`
- `network.rpc.max_open_connections`
- `network.rpc.lag_threshold`
- `network.p2p.persistent_peers`
- `mempool.size`
- `evm.cors_origins`
- `evm.deny_list`
- `logging.level`

---

## 8. Schema Migration

```go
type Migration struct {
    FromVersion int
    ToVersion   int                         // Must equal FromVersion + 1
    Description string
    Migrate     func(cfg *SeiConfig) error  // Must set cfg.Version = ToVersion
}
```

`NewMigrationRegistry(migrations...)` validates migrations form a contiguous chain. `MigrateConfig(cfg, targetVersion)` runs all needed migrations sequentially, then validates the result. Downgrades are rejected.

**Current state:** `DefaultMigrations()` returns an empty slice (v1 is the initial and current schema version). Migrations will be added as the schema evolves.

---

## 9. Embedded Genesis Data

```go
//go:embed chains/*/genesis.json
var chainFS embed.FS
```

Three well-known chains with embedded genesis:

| Chain ID | RPC Endpoint | Genesis Time |
|----------|-------------|--------------|
| `pacific-1` | `https://rpc.sei-apis.com` | 2023-05-22T15:00:00Z |
| `atlantic-2` | `https://rpc-testnet.sei-apis.com` | 2023-02-24T01:00:00Z |
| `arctic-1` | `https://rpc-arctic-1.sei-apis.com` | 2024-01-25T20:18:30Z |

API:
- `KnownChain(chainID) *ChainInfo` -- metadata for known chain, nil if unknown
- `KnownChainIDs() []string` -- all known chain IDs
- `GenesisForChain(chainID) ([]byte, error)` -- embedded genesis.json bytes, error if unknown

Unknown chains must supply their own genesis source (S3 fallback at `{SEI_GENESIS_BUCKET}/{chainID}/genesis.json`).

---

## 10. Relationship to Cosmos SDK Config (config.toml / app.toml)

sei-config **overlays and unifies** -- it does NOT replace the standard Cosmos config files. Instead:

1. `SeiConfig` is a unified superset that covers all fields from both files
2. On disk, the two-file layout is preserved for backward compatibility with existing seid binaries
3. `legacy.go` defines intermediate types that match the exact TOML schemas of `config.toml` and `app.toml`
4. Conversion functions `toLegacyTendermint()`, `toLegacyApp()`, and `fromLegacy()` handle the mapping

Key mapping differences:
- config.toml uses **hyphens** (e.g., `persistent-peers`, `trust-height`)
- app.toml uses **underscores** for EVM/Sei sections (e.g., `http_enabled`, `max_tx_pool_txs`)
- SeiConfig's unified TOML tags use **underscores** consistently (e.g., `persistent_peers`)
- Config.toml `[rpc].laddr` maps to `network.rpc.listen_address`
- Config.toml `[p2p].laddr` maps to `network.p2p.listen_address`
- Config.toml `[instrumentation]` merges with app.toml `[telemetry]` into `MetricsConfig`
- App.toml top-level fields like `minimum-gas-prices`, `pruning`, `halt-height` move into `ChainConfig` and `StorageConfig`
- App.toml `[state-commit].sc-*` fields map to `storage.state_commit.*`
- App.toml `[state-store].ss-*` fields map to `storage.state_store.*`

### Legacy sei metadata

A `[sei]` section in `app.toml` stores `mode` and `version`. This preserves the `archive` mode value which Tendermint's config.toml cannot represent (it only understands validator/full/seed). On read, `[sei].mode` takes precedence over `config.toml`'s mode field.

---

## 11. Sei-Specific Configuration (Not in Upstream Cosmos)

The following sections/fields are Sei-specific extensions that do not exist in standard Cosmos SDK or Tendermint:

### EVM Configuration (`evm.*`)
The entire EVM section -- JSON-RPC HTTP/WS servers, tracing, simulation, worker pools, deny lists, EVM query gas limit, eth replay, block test config.

### SeiDB Storage (`storage.state_commit.*`, `storage.state_store.*`)
- State commit layer with MemIAVL (replaces standard IAVL)
- State store with PebbleDB/RocksDB backends
- Write/Read mode routing between Cosmos and EVM backends

### Giga Executor (`giga_executor.*`)
Parallel execution engine with OCC support.

### Optimistic Concurrency Control
- `chain.occ_enabled`
- `chain.concurrency_workers`

### Light Invariance (`light_invariance.*`)
Supply invariance checks.

### Self-Remediation (`self_remediation.*`)
Automatic restart on P2P peer loss, statesync failures, or falling behind.

### Genesis Stream Import (`genesis.*`)
Stream import for large genesis files.

### Consensus Extensions
- `consensus.gossip_transaction_key_only` -- gossip only tx hashes
- `consensus.unsafe_bypass_commit_timeout_override` -- bypass commit timeout (`*bool` for tri-state)

### P2P Extensions
- `network.p2p.blocksync_peers` -- dedicated block sync peers
- `network.p2p.dial_interval` -- interval between dial attempts

### Mempool Extensions
- `mempool.check_tx_error_blacklist_enabled/threshold` -- automatic peer blacklisting
- `mempool.pending_*` -- pending transaction pool
- `mempool.drop_priority_threshold/utilisation_threshold/reservoir_size` -- priority-based dropping
- `mempool.duplicate_txs_cache_size` -- dedup cache

### RPC Extensions
- `network.rpc.lag_threshold` -- block lag health endpoint
- `network.rpc.timeout_read` -- read timeout

---

## 12. API Surface Summary

### Primary Entry Points

| Function | Use Case |
|----------|----------|
| `DefaultForMode(mode)` | Generate complete config with mode defaults |
| `ResolveIntent(intent)` | Bootstrap: intent -> validated config (sidecar) |
| `ResolveIncrementalIntent(intent, current)` | Day-2 patch: overlay intent onto existing config |
| `ValidateIntent(intent)` | Dry-run validation (controller, before submitting task) |
| `Validate(cfg)` | Validate a fully-formed config |
| `ApplyOverrides(cfg, overrides)` | Apply dotted-key overrides to existing config |
| `ResolveEnv(cfg)` | Apply SEI_/SEID_ env var overrides |
| `ReadConfigFromDir(homeDir)` | Load config.toml + app.toml into SeiConfig |
| `WriteConfigToDir(cfg, homeDir)` | Write SeiConfig as config.toml + app.toml |
| `BuildRegistry()` | Build field metadata registry for introspection |
| `DefaultEnrichments()` | Curated field descriptions/metadata |
| `KnownChain(chainID)` | Look up well-known chain metadata |
| `GenesisForChain(chainID)` | Get embedded genesis.json bytes |
| `SnapshotGenerationOverrides(keepRecent)` | Config overrides for snapshot-producing nodes |

### Controller-Specific Pattern

The controller builds a `ConfigIntent` and passes it to the sidecar. The controller uses `ValidateIntent()` for dry-run validation before task submission. It does not call `DefaultForMode`, `ApplyOverrides`, or `Validate` directly. The sidecar calls `ResolveIntent` or `ResolveIncrementalIntent` to produce the actual config files.

### Pruning Constants

```go
PruningDefault    = "default"
PruningNothing    = "nothing"
PruningEverything = "everything"
PruningCustom     = "custom"
```
