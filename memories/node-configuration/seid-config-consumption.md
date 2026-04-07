---
topic: "seid startup: how the Sei binary consumes configuration"
sources:
  - repo: sei-chain
    version: v0.0.38
    files:
      - cmd/seid/main.go
      - cmd/seid/cmd/root.go
      - app/app.go
      - app/const.go
      - app/params/config.go
      - app/seidb.go
  - repo: sei-tendermint
    version: v0.6.4
    files:
      - config/config.go
      - config/toml.go
      - libs/cli/setup.go
      - node/public.go
      - node/node.go
      - node/setup.go
  - repo: sei-cosmos
    version: v0.3.66
    files:
      - server/start.go
      - server/util.go
      - server/cmd/execute.go
      - server/config/config.go
      - server/config/toml.go
      - client/config/config.go
  - repo: sei-config
    version: v0.0.9-0.20260327015454-7cf35ff77daa
    files:
      - config.go
      - types.go
      - defaults.go
      - io.go
      - legacy.go
      - resolve.go
      - intent.go
      - registry.go
      - enrichments.go
      - validate.go
      - migrate.go
      - chains.go
verified: 2026-04-07
confidence: high
---

# seid Startup: Complete Config Consumption Path

## 1. Startup Sequence (`seid start`)

The full chain from `main()` to a running node:

```
main()
  -> params.SetAddressPrefixes()        // set sei bech32 prefixes
  -> cmd.NewRootCmd()                   // build cobra command tree
  -> svrcmd.Execute(rootCmd, "~/.sei")  // run via Tendermint CLI executor
```

### 1a. Execute (sei-cosmos/server/cmd/execute.go)

```go
func Execute(rootCmd *cobra.Command, defaultHome string) error {
    srvCtx := server.NewDefaultContext()
    ctx = context.WithValue(ctx, server.ServerContextKey, srvCtx)
    executor := tmcli.PrepareBaseCmd(rootCmd, "", defaultHome)
    return executor.ExecuteContext(ctx)
}
```

Key: `PrepareBaseCmd` passes an **empty** env prefix (not "SEI") to the Tendermint CLI layer. The "SEI" prefix is set separately on the client-side Viper instance.

### 1b. PrepareBaseCmd (sei-tendermint/libs/cli/setup.go)

```go
func PrepareBaseCmd(cmd *cobra.Command, envPrefix, defaultHome string) *cobra.Command {
    cobra.OnInitialize(func() { InitEnv(envPrefix) })
    cmd.PersistentFlags().StringP("home", "", defaultHome, "directory for config and data")
    cmd.PersistentPreRunE = concatCobraCmdFuncs(BindFlagsLoadViper, cmd.PersistentPreRunE)
    return cmd
}
```

This prepends `BindFlagsLoadViper` before the root command's existing `PersistentPreRunE`.

### 1c. PersistentPreRunE Chain (sei-chain/cmd/seid/cmd/root.go)

When any command runs, the PersistentPreRunE executes in order:

1. **BindFlagsLoadViper** (from PrepareBaseCmd) -- reads config.toml via the global viper
2. **Root command's PersistentPreRunE** which calls:
   - `client.ReadPersistentCommandFlags()` -- reads CLI flags into client context
   - `config.ReadFromClientConfig()` -- reads `~/.sei/config/client.toml`
   - `server.InterceptConfigsPreRunHandler(cmd, customAppTemplate, customAppConfig)` -- the main config loading

### 1d. InterceptConfigsPreRunHandler (sei-cosmos/server/util.go)

This is the heart of config loading. It:

1. Creates a new `server.Context` with a fresh Viper and default Tendermint config
2. Binds all command flags to Viper
3. Sets env prefix to the **executable basename** (typically "seid")
4. Configures env key replacement: `.` -> `_`, `-` -> `_`
5. Enables `AutomaticEnv()`
6. Calls `interceptConfigs()` to load config files
7. Binds flags again via `bindFlags(basename, cmd, viper)` with `SEID_` env prefix
8. Sets up logging from config

### 1e. interceptConfigs (sei-cosmos/server/util.go) -- THE CRITICAL FUNCTION

```go
func interceptConfigs(rootViper *viper.Viper, customAppTemplate string, customConfig interface{}) (*tmcfg.Config, error) {
    rootDir := rootViper.GetString(flags.FlagHome)       // typically ~/.sei
    configPath := filepath.Join(rootDir, "config")
    tmCfgFile := filepath.Join(configPath, "config.toml")

    conf := tmcfg.DefaultConfig()    // Start with Tendermint defaults

    if config.toml exists:
        rootViper.SetConfigType("toml")
        rootViper.SetConfigName("config")
        rootViper.AddConfigPath(configPath)
        rootViper.ReadInConfig()     // STEP 1: Read config.toml into Viper
    else:
        // Create config.toml from defaults with Sei-specific tweaks
        tmcfg.EnsureRoot(rootDir)
        tmcfg.WriteConfigFile(rootDir, conf)

    rootViper.Unmarshal(conf)        // STEP 2: Unmarshal Viper -> tmcfg.Config struct
    conf.SetRoot(rootDir)

    // Now handle app.toml
    if app.toml does not exist:
        // Create it from customAppConfig (the Sei CustomAppConfig)
        config.WriteConfigFile(appCfgFilePath, customConfig)

    rootViper.SetConfigType("toml")
    rootViper.SetConfigName("app")
    rootViper.AddConfigPath(configPath)
    rootViper.MergeInConfig()        // STEP 3: Merge app.toml INTO SAME Viper

    return conf, nil
}
```

## 2. Config Files and Directory Structure

### `~/.sei/` directory layout

```
~/.sei/
├── config/
│   ├── config.toml          # Tendermint config (P2P, RPC, consensus, mempool, etc.)
│   ├── app.toml             # Cosmos SDK + Sei extensions (gas prices, pruning, EVM, WASM, SeiDB, etc.)
│   ├── client.toml          # Client-side config (chain-id, keyring-backend, node RPC endpoint)
│   ├── genesis.json         # Genesis document (chain state at height 0)
│   ├── node_key.json        # P2P identity key (ed25519)
│   └── priv_validator_key.json  # Validator signing key (only for validators)
├── data/
│   ├── application.db/      # Application state LevelDB
│   ├── blockstore.db/       # Block storage
│   ├── state.db/            # Tendermint state (consensus state, validators)
│   ├── evidence.db/         # Evidence of misbehavior
│   ├── tx_index.db/         # Transaction index
│   ├── peerstore.db/        # P2P peer address book
│   ├── cs.wal/              # Consensus WAL (write-ahead log)
│   ├── priv_validator_state.json  # Validator signing state (last signed height/round/step)
│   └── snapshots/           # State sync snapshots
└── wasm/                    # CosmWasm contract code cache
```

### config.toml (Tendermint layer)

Parsed by sei-tendermint. Uses **hyphens** in key names:
- Top-level: `proxy-app`, `moniker`, `mode`, `db-backend`, `db-dir`, `log-level`, `log-format`, `genesis-file`, `node-key-file`, `abci`, `filter-peers`
- `[rpc]`: `laddr`, `cors-allowed-origins`, `max-open-connections`, `unsafe`, etc.
- `[p2p]`: `laddr`, `external-address`, `persistent-peers`, `bootstrap-peers`, `blocksync-peers`, `max-connections`, `pex`, `send-rate`, `recv-rate`, `unconditional-peer-ids`, `private-peer-ids`, `allow-duplicate-ip`, `flush-throttle-timeout`, `max-packet-msg-payload-size`, `handshake-timeout`, `dial-timeout`, `queue-type`
- `[mempool]`: `size`, `max-txs-bytes`, `max-tx-bytes`, `ttl-duration`, `ttl-num-blocks`, `broadcast`, `cache-size`, `check-tx-error-blacklist-enabled`, `pending-size`, etc.
- `[statesync]`: `enable`, `rpc-servers`, `trust-height`, `trust-hash`, `trust-period`, `use-p2p`, etc.
- `[consensus]`: `wal-file`, `create-empty-blocks`, `gossip-tx-key-only`, `unsafe-propose-timeout-override`, `unsafe-commit-timeout-override`, `unsafe-bypass-commit-timeout-override`, etc.
- `[tx-index]`: `indexer`, `psql-conn`
- `[instrumentation]`: `prometheus`, `prometheus-listen-addr`, `max-open-connections`, `namespace`
- `[priv-validator]`: `key-file`, `state-file`, `laddr`
- `[self-remediation]`: `p2p-no-peers-available-window-seconds`, `blocks-behind-threshold`, `restart-cooldown-seconds`, etc.
- `[db-sync]`: `enable`

### app.toml (Cosmos SDK + Sei extensions)

Parsed by sei-cosmos. Mix of hyphens and underscores:
- Top-level (hyphens): `minimum-gas-prices`, `pruning`, `pruning-keep-recent`, `pruning-interval`, `halt-height`, `halt-time`, `min-retain-blocks`, `inter-block-cache`, `iavl-disable-fastnode`, `compaction-interval`, `concurrency-workers`, `occ-enabled`
- `[telemetry]` (hyphens): `enabled`, `service-name`, `prometheus-retention-time`, `global-labels`
- `[api]` (hyphens): `enable`, `swagger`, `address`, `max-open-connections`
- `[grpc]` / `[grpc-web]` (hyphens): `enable`, `address`
- `[state-sync]` (hyphens): `snapshot-interval`, `snapshot-keep-recent`, `snapshot-directory`
- `[state-commit]` (hyphens): `sc-enable`, `sc-directory`, `sc-async-commit-buffer`
- `[state-store]` (hyphens): `ss-enable`, `ss-db-directory`, `ss-backend`, `ss-keep-recent`
- `[evm]` (**underscores**): `http_enabled`, `http_port`, `ws_enabled`, `ws_port`, `simulation_gas_limit`, `cors_origins`, `deny_list`, `max_log_no_block`
- `[wasm]` (underscores): `query_gas_limit`, `lru_size`
- `[eth_replay]` (underscores): `eth_replay_enabled`, `eth_rpc`
- `[giga_executor]` (underscores): `enabled`, `occ_enabled`
- `[genesis]` (hyphens): `stream-import`, `genesis-stream-file`

### client.toml

Read during `PersistentPreRunE` via `ReadFromClientConfig()`:
- `chain-id`: The network chain ID (critical -- validated against genesis)
- `keyring-backend`: `os`, `file`, or `test`
- `output`: `text` or `json`
- `node`: Tendermint RPC endpoint (default `tcp://localhost:26657`)
- `broadcast-mode`: `sync`, `async`, or `block`

### genesis.json

Read by Tendermint's node.New() via `genesisDocProvider`. Contains:
- `chain_id`: Chain identifier
- `genesis_time`: Genesis timestamp
- `initial_height`: Starting block height
- `consensus_params`: Block size, evidence, validator params
- `validators`: Initial validator set
- `app_state`: Module-specific genesis state (bank balances, staking, etc.)

Validated during startup: `genesisFile.ChainID != clientCtx.ChainID` causes a panic.

## 3. Config Source Loading Order and Merge Semantics

The config loading follows this precedence (lowest to highest):

### For Tendermint config (config.toml -> `tmcfg.Config`):

1. **tmcfg.DefaultConfig()** -- hardcoded Go defaults
2. **config.toml file** -- read via `rootViper.ReadInConfig()`
3. **rootViper.Unmarshal(conf)** -- Viper merges env vars and flags on top of file values
4. **SetTendermintConfigs(config)** -- called by `InitCmd` at init time only (NOT at runtime), sets Sei-specific P2P/mempool/consensus defaults for newly created configs
5. **Environment variables** -- via Viper's `AutomaticEnv()` with `SEID_` prefix
6. **CLI flags** -- bound via `BindPFlags` (highest precedence)

### For app config (app.toml -> Viper -> individual `appOpts.Get()` calls):

1. **serverconfig.DefaultConfig()** -- Cosmos SDK defaults
2. **initAppConfig()** overrides (sei-chain/cmd/seid/cmd/root.go):
   - `MinGasPrices = "0.02usei"`
   - `API.Enable = true`
   - `Pruning = "default"`
   - `PruningInterval = random prime between 2500-4000`
   - `Telemetry.Enabled = true`
   - Plus EVM, WASM, ETHReplay, ETHBlockTest, EVMQuery defaults
3. **app.toml file** -- merged via `rootViper.MergeInConfig()` into the same Viper
4. **Environment variables** -- via `AutomaticEnv()` with `SEID_` prefix
5. **CLI flags** -- bound to Viper, highest precedence

### Critical detail: SINGLE VIPER INSTANCE

Both config.toml and app.toml are loaded into the **same** Viper instance. config.toml is loaded first with `ReadInConfig()`, then app.toml is loaded with `MergeInConfig()`. If there are key collisions between the two files, app.toml values win because `MergeInConfig` overwrites existing keys.

The Tendermint `*Config` struct is populated by `Unmarshal` **before** app.toml is merged, so the tmcfg.Config struct only reflects config.toml + env + flags.

The app config is consumed differently -- not via Unmarshal, but via individual `viper.Get*()` calls in `GetConfig()` and in the `newApp()` function via `appOpts.Get()`, which reads from the fully-merged Viper (config.toml + app.toml + env + flags).

## 4. Where sei-config Fits

**sei-config is NOT a dependency of seid.** Neither sei-chain v0.0.38 nor the monorepo version imports it.

sei-config is a **companion library** used by the sidecar and controller to:

1. **Generate** config.toml and app.toml files that seid will consume
2. **Read** existing config.toml and app.toml into a unified `SeiConfig` struct
3. **Validate** configuration before writing
4. **Apply overrides** via the `ConfigIntent` mechanism
5. **Resolve environment variables** with `SEI_` prefix (via `ResolveEnv`)

The pipeline when the sidecar bootstraps config:

```
ConfigIntent (from controller CRD)
  -> ResolveIntent() -- produces SeiConfig from mode defaults + overrides
  -> ResolveEnv()    -- applies SEI_* environment variables
  -> WriteConfigToDir() -- splits SeiConfig into config.toml + app.toml
     -> toLegacyTendermint() -- produces config.toml (hyphen keys)
     -> toLegacyApp()        -- produces app.toml (mixed keys)
```

Then seid reads these files through its normal Viper-based loading.

sei-config's `WriteConfigToDir` uses atomic writes (temp file + rename) to prevent corruption.

### sei-config Key Conversion

sei-config uses **underscores** in its unified TOML schema (e.g., `network.p2p.persistent_peers`), but when writing the legacy config.toml, it converts to **hyphens** (e.g., `persistent-peers`) via the `legacyTendermintConfig` struct tags.

## 5. Viper Wiring Details

### Environment Variable Handling

There are **three** separate Viper configurations in play:

1. **Global Viper** (from `BindFlagsLoadViper` in tendermint/libs/cli/setup.go):
   - Env prefix: empty string (passed as `""` from Execute)
   - Reads config.toml from `~/.sei/config/`
   - Used for the initial Tendermint config bootstrap

2. **Server Context Viper** (from `InterceptConfigsPreRunHandler`):
   - Env prefix: basename of executable (typically `seid`)
   - Key replacer: `.` -> `_`, `-` -> `_`
   - `AutomaticEnv()` enabled
   - Reads config.toml then merges app.toml
   - Environment variables: `SEID_<KEY>` where KEY has dots/hyphens replaced with underscores
   - Example: `SEID_MINIMUM_GAS_PRICES=0.01usei`
   - This is the Viper used for all server-side config access

3. **Client Context Viper** (from `WithViper("SEI")` in root.go):
   - Env prefix: `SEI`
   - Used for client-side config (chain-id, keyring, etc.)
   - Environment variables: `SEI_<KEY>`

### Flag Binding (bindFlags in sei-cosmos/server/util.go)

```go
func bindFlags(basename string, cmd *cobra.Command, v *viper.Viper) {
    cmd.Flags().VisitAll(func(f *pflag.Flag) {
        v.BindEnv(f.Name, fmt.Sprintf("%s_%s", basename, strings.ToUpper(strings.ReplaceAll(f.Name, "-", "_"))))
        v.BindPFlag(f.Name, f)
        // If flag not explicitly set but Viper has a value (from config/env), set the flag
        if !f.Changed && v.IsSet(f.Name) {
            cmd.Flags().Set(f.Name, fmt.Sprintf("%v", v.Get(f.Name)))
        }
    })
}
```

This creates explicit env var bindings like `SEID_MINIMUM_GAS_PRICES` for the `--minimum-gas-prices` flag.

## 6. Config Flow to Node Components

### P2P Layer

```
config.toml -> Viper -> Unmarshal -> tmcfg.Config.P2P
  -> node.New(cfg) -> createPeerManager(cfg)
     -> cfg.P2P.PersistentPeers -> parsed into p2p.NodeAddress list
     -> cfg.P2P.BootstrapPeers  -> parsed into p2p.NodeAddress list
     -> cfg.P2P.BlockSyncPeers  -> parsed into p2p.NodeAddress list
     -> cfg.P2P.MaxConnections  -> PeerManagerOptions.MaxConnected
     -> cfg.P2P.PrivatePeerIDs  -> PeerManagerOptions.PrivatePeers
     -> cfg.P2P.UnconditionalPeerIDs -> PeerManagerOptions.UnconditionalPeers
     -> cfg.P2P.ExternalAddress -> PeerManagerOptions.SelfAddress
  -> createRouter(cfg) -> uses cfg.P2P for transport settings
```

### Consensus

```
tmcfg.Config.Consensus -> passed to consensus.NewReactor()
  -> UnsafeProposeTimeoutOverride, UnsafeCommitTimeoutOverride, etc.
  -> CreateEmptyBlocks, GossipTransactionKeyOnly
```

### Application Layer

```
app.toml -> Viper (merged) -> appOpts (servertypes.AppOptions interface)
  -> newApp(logger, db, traceStore, tmConfig, appOpts)
     -> appOpts.Get("minimum-gas-prices") -> baseapp.SetMinGasPrices()
     -> appOpts.Get("pruning") -> baseapp.SetPruning()
     -> appOpts.Get("halt-height") -> baseapp.SetHaltHeight()
     -> appOpts.Get("sc-enable") -> SetupSeiDB() for MemIAVL/PebbleDB
     -> appOpts.Get("occ-enabled") -> baseapp.SetOccEnabled()
     -> cast.ToUint64(appOpts.Get("state-sync.snapshot-interval"))
```

## 7. Sei-Specific Config Overrides at Init Time

When `seid init` creates a new node, `SetTendermintConfigs` (in app/params/config.go) applies these Sei-tuned defaults:

```go
func SetTendermintConfigs(config *tmcfg.Config) {
    config.P2P.MaxConnections = 200
    config.P2P.SendRate = 20480000
    config.P2P.RecvRate = 20480000
    config.P2P.MaxPacketMsgPayloadSize = 1000000
    config.P2P.FlushThrottleTimeout = 10 * time.Millisecond
    config.Mempool.Size = 1000
    config.Mempool.MaxTxsBytes = 10737418240
    config.Mempool.MaxTxBytes = 2048576
    config.Mempool.TTLDuration = 30 * time.Second
    config.Mempool.TTLNumBlocks = 100
    config.Consensus.GossipTransactionKeyOnly = true
    config.Consensus.UnsafeProposeTimeoutOverride = 300 * time.Millisecond
    config.Consensus.UnsafeProposeTimeoutDeltaOverride = 50 * time.Millisecond
    config.Consensus.UnsafeVoteTimeoutOverride = 50 * time.Millisecond
    config.Consensus.UnsafeVoteTimeoutDeltaOverride = 50 * time.Millisecond
    config.Consensus.UnsafeCommitTimeoutOverride = 200 * time.Millisecond
    config.Consensus.UnsafeBypassCommitTimeoutOverride = &false
    config.Instrumentation.Prometheus = true
}
```

These are written to config.toml at init and then read back on subsequent starts.

## 8. Environment Variable Override Patterns

### Via seid's Viper (at runtime)

The server-side Viper uses the executable basename as prefix:

| Config Key | Env Var | Source File |
|---|---|---|
| `minimum-gas-prices` | `SEID_MINIMUM_GAS_PRICES` | app.toml |
| `pruning` | `SEID_PRUNING` | app.toml |
| `p2p.persistent-peers` | `SEID_P2P_PERSISTENT_PEERS` | config.toml |
| `statesync.enable` | `SEID_STATESYNC_ENABLE` | config.toml |
| `api.enable` | `SEID_API_ENABLE` | app.toml |

### Via sei-config's ResolveEnv (sidecar pre-processing)

sei-config uses its own env var resolution with `SEI_` and legacy `SEID_` prefixes:

| sei-config Key | Env Var |
|---|---|
| `network.p2p.persistent_peers` | `SEI_NETWORK_P2P_PERSISTENT_PEERS` |
| `state_sync.trust_height` | `SEI_STATE_SYNC_TRUST_HEIGHT` |
| `storage.pruning` | `SEI_STORAGE_PRUNING` |
| `evm.http_port` | `SEI_EVM_HTTP_PORT` |

Legacy `SEID_` prefix vars are also recognized with lower precedence and generate deprecation warnings.

**These are two separate systems.** sei-config env vars are resolved by the sidecar before writing files; seid Viper env vars are resolved at seid startup from the merged config.

## 9. Invalid or Missing Configuration Handling

### Missing config.toml
If `~/.sei/config/config.toml` doesn't exist, `interceptConfigs` creates the directory structure and writes a default config.toml with Sei-specific tweaks (RPC pprof on localhost:6060, higher P2P rates, 5s commit timeout).

### Missing app.toml
If `~/.sei/config/app.toml` doesn't exist, it's generated from the `CustomAppConfig` template defined in `initAppConfig()` with Sei defaults.

### Missing client.toml
If `~/.sei/config/client.toml` doesn't exist, `ReadFromClientConfig` creates it with defaults.

### Chain ID mismatch
During `seid start`, a panic occurs if:
- `--chain-id` flag doesn't match `client.toml`'s `chain-id`
- `genesis.json`'s `chain_id` doesn't match `client.toml`'s `chain-id`

### Invalid config values
- sei-tendermint's `Config.ValidateBasic()` validates ranges (e.g., mempool size > 0, consensus timeouts > 0)
- sei-cosmos's `Config.ValidateBasic()` checks min-gas-prices format
- Invalid persistent peer addresses cause `createPeerManager` to fail with a specific error
- The node **panics** on chain-id mismatches; returns errors for most other validation failures

### Empty minimum-gas-prices
If `minimum-gas-prices` is empty in app.toml, the SDK logs a warning but allows startup (defaults to 0). Future SDK versions will make this an error.

## 10. Parameters Only Settable via Environment Variables

There are no config parameters in seid that can **only** be set via environment variables. All Viper-managed settings can be set via config files or CLI flags. However:

- Some flags have **no config file equivalent** (e.g., `--cpu-profile`, `--trace-store`, `--grpc-only`). These are runtime-only flags.
- The `SEI_` prefix env vars processed by sei-config's `ResolveEnv()` operate on the `SeiConfig` struct and are translated to file values. They don't have a direct seid runtime equivalent -- they pre-process config before seid reads it.

## 11. Summary: Complete Config Loading Timeline

```
1. main() -> SetAddressPrefixes, NewRootCmd, Execute
2. Execute -> PrepareBaseCmd("", "~/.sei") -> registers BindFlagsLoadViper as pre-hook
3. User runs `seid start`
4. BindFlagsLoadViper (tendermint layer):
   - Bind flags to global viper
   - Set viper config name = "config", path = ~/.sei/config/
   - ReadInConfig() -> reads config.toml (if exists)
5. Root PersistentPreRunE:
   a. ReadPersistentCommandFlags -> CLI flags into client context
   b. ReadFromClientConfig -> reads client.toml, gets chain-id
   c. InterceptConfigsPreRunHandler(customAppTemplate, customAppConfig):
      i.   Create fresh server Viper
      ii.  Bind all flags to server Viper
      iii. Set env prefix = "seid", key replacer (.-_), AutomaticEnv
      iv.  interceptConfigs():
           - Create/read config.toml -> Viper
           - Unmarshal Viper -> tmcfg.Config
           - Create/read app.toml -> Merge into same Viper
      v.   bindFlags("seid", cmd, viper) -> SEID_* env var bindings
      vi.  Set up logging from config
6. StartCmd.PreRunE:
   - Bind start-specific flags to server Viper
   - Validate pruning options
7. StartCmd.RunE:
   - Read chain-id from client config, validate against flag and genesis
   - config.GetConfig(viper) -> parse app config from merged Viper
   - appCreator(logger, db, traceWriter, tmConfig, viper) -> create Sei App
   - node.New(ctx, tmConfig, ...) -> create Tendermint node
     - Load/gen node key
     - Load/gen priv validator key
     - makeNode(cfg, ...) -> create all reactors
       - createPeerManager(cfg) -> P2P from cfg.P2P
       - createRouter -> transport layer
       - consensus, mempool, statesync, blocksync reactors
   - tmNode.Start() -> node begins operation
   - Start API server, gRPC server, gRPC-Web, Rosetta
   - Wait for shutdown signal
```
