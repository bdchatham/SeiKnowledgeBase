---
topic: "seictl: CLI tool for Sei node management"
sources:
  - repo: seictl
    version: v0.0.23
    files:
      - CLAUDE.md
      - README.md
      - main.go
      - go.mod
      - version.json
      - config.go
      - genesis.go
      - patch.go
      - serve.go
      - await.go
      - outputter.go
      - Dockerfile
      - Makefile
      - .goreleaser.yaml
      - internal/patch/merge.go
      - internal/patch/file.go
      - internal/patch/toml.go
      - internal/patch/json.go
      - sidecar/engine/engine.go
      - sidecar/engine/types.go
      - sidecar/engine/cron.go
      - sidecar/server/server.go
      - sidecar/client/client.go
      - sidecar/client/tasks.go
      - sidecar/api/openapi.yaml
      - sidecar/rpc/status.go
      - sidecar/s3/client.go
      - sidecar/actions/process.go
      - sidecar/tasks/cmd.go
      - sidecar/tasks/config.go
      - sidecar/tasks/config_apply.go
      - sidecar/tasks/config_reload.go
      - sidecar/tasks/config_validate.go
      - sidecar/tasks/genesis.go
      - sidecar/tasks/genesis_peers.go
      - sidecar/tasks/generate_identity.go
      - sidecar/tasks/generate_gentx.go
      - sidecar/tasks/assemble_genesis.go
      - sidecar/tasks/upload_genesis_artifacts.go
      - sidecar/tasks/peers.go
      - sidecar/tasks/statesync.go
      - sidecar/tasks/snapshot_restore.go
      - sidecar/tasks/snapshot_upload.go
      - sidecar/tasks/result_export.go
      - sidecar/tasks/await_condition.go
      - sidecar/tasks/ready.go
      - sidecar/tasks/tendermint.go
      - sidecar/tasks/defaults/defaults.go
      - sidecar/tasks/defaults/config.toml
verified: 2026-04-07
confidence: high
---

# seictl

## Purpose and Identity

seictl is a **dual-purpose Go binary** that serves two distinct roles:

1. **CLI tool** for Sei node operators -- provides commands to patch configuration files (TOML) and genesis files (JSON) on disk.
2. **Sidecar HTTP server** for the sei-k8s-controller -- runs as a container alongside seid in Kubernetes, exposing a task-based HTTP API that the controller drives to bootstrap and manage nodes.

It is packaged as `ghcr.io/sei-protocol/seictl` via Docker (distroless base) and distributed as native binaries via GoReleaser for Linux (x86_64, arm64, armv7), macOS (Intel, Apple Silicon), and Windows (x86_64).

## Complete Command Tree

```
seictl [--home <path>]
  |
  +-- config [--target app|client|config]
  |     +-- patch [--output|-o <path>] [--in-place-rewrite|-i] [file]
  |
  +-- genesis
  |     +-- patch [--output|-o <path>] [--in-place-rewrite|-i] [file]
  |
  +-- patch --target <file-path> [--output|-o <path>] [--in-place-rewrite|-i] [file]
  |
  +-- await [--timeout <duration>]
  |     +-- validator [--api <url>] <address>
  |
  +-- serve [--port <port>]
```

### Global Flags
- `--home <path>`: Sei home directory (default: `~/.sei`, env: `SEI_HOME`).

### CLI Commands (operator-facing)

#### `config patch`
Applies a TOML merge-patch to one of three seid config files: `app.toml`, `client.toml`, or `config.toml`.

- **Auto-detection**: When `--target` is omitted, seictl inspects the top-level keys in the patch and matches them against a hardcoded hints table (`configTargetHints`) that maps every known seid config key to its file. If keys span multiple files, it refuses (safety).
- **Input**: Reads patch from a file argument or stdin.
- **Output**: Stdout by default, or `--output <path>`, or `--in-place-rewrite` (atomic write via temp file + rename).
- **Merge algorithm**: Recursive merge-patch -- nested maps are merged, `null` deletes keys, scalars are replaced.

#### `genesis patch`
Applies a JSON merge-patch to `$HOME/.sei/config/genesis.json`. Same input/output mechanics as `config patch`.

#### `patch`
Universal merge-patch for any TOML or JSON file (not Sei-specific). Requires `--target <file-path>`. Determines format from file extension (`.toml` or `.json`).

#### `await validator <address>`
Polls the Sei HTTP API (`/cosmos/staking/v1beta1/validators/{address}`) until the validator is found on-chain or the timeout is reached. Used during genesis ceremonies to confirm validator registration.

- API URL auto-discovered from `app.toml` `[api].address` field, or explicitly via `--api`.
- Default timeout: 1 minute. Configurable via `--timeout`.
- Backoff: 1 second between attempts, 5 second per-attempt timeout.

### Sidecar Command (controller-facing)

#### `serve`
Starts the sidecar HTTP API and task engine. This is the primary mode when running as a Kubernetes sidecar container.

- **Port**: 7777 (default, env: `SEI_SIDECAR_PORT`).
- **Home directory**: Uses `--home` flag, defaults to `/sei` (not `~/.sei`) in serve mode.
- **Startup**: Calls `EnsureDefaultConfig(homeDir)` which creates the directory structure (`config/`, `data/`) and writes an embedded default `config.toml` if none exists.
- **Scheduler**: A background goroutine runs `EvalSchedules()` every 10 seconds to fire cron-scheduled tasks.

## Sidecar HTTP API

Defined in `sidecar/api/openapi.yaml` (v0.6.0). Base URL: `http://localhost:7777`.

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v0/healthz` | Returns 200 after mark-ready task completes; 503 otherwise. Used as k8s readiness probe. |
| GET | `/v0/status` | Returns `{"status": "Initializing"}` or `{"status": "Ready"}`. |
| GET | `/v0/node-id` | Returns `{"nodeId": "<hex>"}` -- the CometBFT node ID derived from `node_key.json`. Not in OpenAPI spec. |
| POST | `/v0/tasks` | Submit a task (one-shot or scheduled). Returns `{"id": "<uuid>"}`. |
| GET | `/v0/tasks` | List recent task results (max 10), active tasks, and scheduled tasks. |
| GET | `/v0/tasks/{id}` | Get a single task result by UUID. |
| DELETE | `/v0/tasks/{id}` | Remove/cancel a task. |

### Task Request Format
```json
{
  "id": "optional-uuid",
  "type": "task-type-string",
  "params": { ... },
  "schedule": { "cron": "*/5 * * * *" }
}
```

When `id` is provided, the engine uses it as the canonical identifier (enabling deterministic/idempotent submissions from the controller). When a task with the same ID already exists (active or completed), the existing ID is returned without re-submitting.

When `schedule` is provided, the task recurs on the cron. Otherwise it runs once immediately.

## Task Engine

Located in `sidecar/engine/`. The engine is a concurrent task executor:

- Each submitted task runs in its own goroutine.
- Maintains a ring buffer of the last 10 completed results.
- Active tasks can be cancelled via DELETE.
- Scheduled tasks fire on cron intervals, evaluated every 10 seconds.
- Overlap guard prevents a scheduled task from running concurrently with itself.
- Ready state is set when a `mark-ready` task completes successfully.

## Complete Task Type Catalog

### Bootstrap / Initialization Tasks

#### `generate-identity`
Creates the validator identity files -- the same operation as `seid init`:
- Calls `genutil.InitializeNodeValidatorFilesFromMnemonic` to generate `node_key.json`, `priv_validator_key.json`, `priv_validator_state.json`.
- Writes `config.toml` via CometBFT SDK.
- Creates a minimal `genesis.json` if one does not exist.
- Params: `chainId`, `moniker`.
- Idempotent via marker file `.sei-sidecar-identity-done`.

#### `generate-gentx`
Produces a genesis transaction (gentx) -- equivalent to `seid keys add` + `seid add-genesis-account` + `seid gentx`:
- Creates a validator key in a test keyring.
- Adds the validator account and balance to genesis (auth + bank modules).
- Derives the Ethereum address and adds an EVM address association (avoids importing x/evm directly -- operates on genesis JSON to stay CGO_ENABLED=0).
- Builds, signs, and writes a `MsgCreateValidator` gentx file.
- Params: `chainId`, `stakingAmount`, `accountBalance`.
- Idempotent via marker file `.sei-sidecar-gentx-done`.

#### `upload-genesis-artifacts`
Uploads gentx and identity manifests to S3 for the assembler to collect:
- Uploads `<prefix>/<nodeName>/gentx.json`.
- Uploads `<prefix>/<nodeName>/identity.json` (contains `node_key.json`).
- Params: `s3Bucket`, `s3Prefix`, `s3Region`, `nodeName`.
- Idempotent via marker file `.sei-sidecar-artifact-upload-done`.

#### `assemble-and-upload-genesis`
The genesis ceremony coordinator -- equivalent to `seid collect-gentxs`:
1. Downloads each node's gentx from S3.
2. Adds any missing genesis accounts (each node only added its own during `generate-gentx`).
3. Calls `genutil.GenAppStateFromConfig` -- the exact SDK function behind `seid collect-gentxs` -- to produce the final genesis.
4. Uploads assembled `genesis.json` to S3.
5. Builds `peers.json` from each node's `identity.json` using Kubernetes DNS addresses (`<name>-0.<name>.<namespace>.svc.cluster.local:26656`).
6. Uploads `peers.json` to S3.
- Params: `s3Bucket`, `s3Prefix`, `s3Region`, `chainId`, `namespace`, `accountBalance`, `nodes` (list of `{name: "..."}` objects).
- Idempotent via marker file `.sei-sidecar-assemble-done`.

#### `configure-genesis`
Writes genesis.json to the config directory:
- **S3 path**: When `uri` (s3://bucket/key) and `region` are provided, downloads from S3.
- **Embedded path**: When no S3 params, writes the embedded genesis for the configured chain ID (from `SEI_CHAIN_ID` env var) using `seiconfig.GenesisForChain()`.
- Idempotent via marker file `.sei-sidecar-genesis-done`.

#### `set-genesis-peers`
Downloads `peers.json` from S3 (produced by the assembler), filters out the current node's own entry, and writes the remaining entries to `config.toml` as `persistent-peers`.
- Params: `s3Bucket`, `s3Key`, `s3Region`.

#### `mark-ready`
A no-op task whose successful completion marks the engine as "Ready" (healthz starts returning 200). Signals that bootstrap is complete.

### Configuration Tasks

#### `config-patch`
Applies generic TOML merge-patches to one or more seid config files:
```json
{
  "files": {
    "config.toml": {"p2p": {"persistent-peers": "..."}},
    "app.toml":    {"pruning": "nothing"}
  }
}
```
Each file is read, merge-patched, and atomically rewritten.

#### `config-apply`
Generates or patches node config using **sei-config**'s intent resolution pipeline:
- **Full mode**: Resolves a `ConfigIntent` (mode + targetVersion + overrides) from mode defaults, writes all config files.
- **Incremental mode**: Reads current on-disk config, applies overrides incrementally.
- Uses `seiconfig.ResolveIntent()` / `seiconfig.ResolveIncrementalIntent()`.
- Diagnostics are returned as structured JSON errors if validation fails.
- Params: `mode`, `targetVersion`, `incremental` (bool), `overrides` (map[string]string).

#### `config-validate`
Reads on-disk config and validates it via `seiconfig.Validate()`. Returns structured diagnostics JSON as an error if validation fails.

#### `config-reload`
Patches hot-reloadable fields on disk:
- Validates that all requested fields are marked `HotReload: true` in the sei-config field registry.
- Applies overrides via `seiconfig.ApplyOverrides()`.
- Validates the resulting config.
- Writes to disk.
- **Note**: Signaling seid to re-read config (SIGHUP or API call) is not yet implemented (marked as TODO).
- Params: `fields` (map[string]string).

### Networking Tasks

#### `discover-peers`
Resolves peers from multiple sources and writes them to `config.toml` as `persistent-peers`:

**Source types:**
- `ec2Tags`: Queries AWS EC2 DescribeInstances for running instances matching tag filters. For each instance, queries its CometBFT RPC (`http://<ip>:26657/status`) to get the node ID. Builds `nodeId@ip:26656` peer strings. Prefers public IP, falls back to private.
- `static`: Returns a fixed list of pre-configured peer addresses.

Params: `sources` (array of source objects).

#### `configure-state-sync`
Discovers a trust point from peers and configures CometBFT state sync in `config.toml`:
1. Reads `persistent-peers` from config.toml.
2. Extracts RPC host addresses from peer strings.
3. Either queries the latest height from a peer's RPC (`/status`) and subtracts 2000 for the trust height, or uses the locally-restored snapshot height.
4. Queries the block hash at the trust height (`/block?height=N`).
5. Writes `[statesync]` section: `enable=true`, `trust-height`, `trust-hash`, `rpc-servers`, `trust-period`, `use-local-snapshot`, `backfill-blocks`.
- Idempotent via marker file `.sei-sidecar-statesync-done`.

### Snapshot Tasks

#### `snapshot-restore`
Downloads and extracts a snapshot archive from S3:
1. Reads `<prefix>latest.txt` from S3 to get the current snapshot height.
2. Constructs the snapshot key: `<prefix>snapshot_<height>_<chainId>_<region>.tar.gz`.
3. Downloads the archive using the S3 transfer manager (parallel byte-range downloads).
4. Extracts the tar.gz to `<homeDir>/data/snapshots/`.
5. Records the snapshot height to `.sei-sidecar-snapshot-height` for downstream tasks.
- Params: `bucket`, `prefix`, `region`, `chainId`.
- Idempotent via marker file `.sei-sidecar-snapshot-done`.

#### `snapshot-upload`
Archives and streams locally-produced Tendermint state-sync snapshots to S3:
1. Scans `<homeDir>/data/snapshots/` for snapshot height directories.
2. Picks the **second-to-latest** height (avoids uploading an in-progress snapshot).
3. Checks against a local state file (`.sei-sidecar-last-upload.json`) to skip already-uploaded heights.
4. Streams a tar.gz archive directly to S3 via `io.Pipe` (no full in-memory buffering).
5. Updates `latest.txt` in S3 with the uploaded height.
- Params: `bucket`, `prefix`, `region`.
- Designed to be scheduled on a cron.

### Data Export Tasks

#### `result-export`
Queries the local seid RPC for block results and uploads them as compressed NDJSON pages to S3:
1. Reads export state from `.sei-sidecar-last-export.json` (bootstraps from snapshot height if no state file exists).
2. Queries `/status` for the latest height.
3. Exports in pages of 1000 blocks: queries `/block_results?height=N` for each height.
4. Each page is streamed as gzipped NDJSON to S3 (`<prefix><start>-<end>.ndjson.gz`).
5. Tracks progress across invocations via the state file.
- Designed to be scheduled on a cron.
- Params: `bucket`, `prefix`, `region`, `rpcEndpoint` (defaults to localhost:26657).
- Graceful degradation: logs warnings and defers to next scheduled run on failures.

### Lifecycle Tasks

#### `await-condition`
Polls the local CometBFT RPC until a condition is met, then optionally executes a post-condition action:
- **Conditions**: `height` -- polls `/status` until `latest_block_height >= targetHeight`.
- **Actions**: `SIGTERM_SEID` -- sends SIGTERM to the seid process (via `/proc` scanning, requires `shareProcessNamespace: true` in the pod spec). Escalates to SIGKILL after 30 seconds.
- Params: `condition`, `targetHeight`, `action` (optional).

## How seictl Interacts with Running Sei Nodes

seictl interacts with seid through multiple channels:

1. **Filesystem**: Directly reads and writes seid's config files (`config.toml`, `app.toml`, `client.toml`, `genesis.json`). The sidecar container shares the same volume as seid.

2. **CometBFT RPC (port 26657)**: The sidecar queries the local node's `/status`, `/block`, and `/block_results` endpoints for height information, block hashes, and result export.

3. **Sei HTTP API (port 1317)**: The `await validator` CLI command polls the Cosmos REST API.

4. **Process signals**: The `await-condition` task can send SIGTERM/SIGKILL to the seid process via `/proc` scanning (requires shared process namespace in k8s).

5. **Cosmos SDK libraries**: The `generate-identity` and `generate-gentx` tasks call the same Go SDK functions as `seid init`, `seid keys add`, `seid add-genesis-account`, `seid gentx`, and `seid collect-gentxs` -- all in-process, without shelling out.

## Relationship to sei-config

seictl depends on `github.com/sei-protocol/sei-config` (v0.0.9-prerelease) for:

- **Intent resolution**: `config-apply` uses `seiconfig.ResolveIntent()` and `seiconfig.ResolveIncrementalIntent()` to generate config from high-level intents (mode + version + overrides).
- **Config I/O**: `seiconfig.ReadConfigFromDir()` and `seiconfig.WriteConfigToDir()` for structured config read/write.
- **Validation**: `seiconfig.Validate()` for config validation, `seiconfig.ValidateIntent()` for intent validation.
- **Field registry**: `seiconfig.BuildRegistry()` to check which fields are hot-reloadable.
- **Overrides**: `seiconfig.ApplyOverrides()` for applying field-level changes.
- **Embedded genesis**: `seiconfig.GenesisForChain()` for well-known chain genesis files.
- **Port constants**: `seiconfig.PortRPC` for default RPC endpoint construction.

## Relationship to sei-chain

seictl depends on `github.com/sei-protocol/sei-chain` (v0.0.29-fix prerelease) for:

- **CometBFT config**: `tmcfg.DefaultConfig()`, `tmcfg.EnsureRoot()`, `tmcfg.WriteConfigFile()`.
- **Genesis types**: `tmtypes.GenesisDoc`, `genutil.ExportGenesisFile()`.
- **Key management**: `keyring.New()`, `hd.Secp256k1`, key generation and signing.
- **Account types**: `authtypes`, `banktypes`, `stakingtypes` for genesis state manipulation.
- **Transaction building**: `tx.Factory`, `stakingcli.BuildCreateValidatorMsg()`, `authclient.SignTx()`.
- **Genesis validation**: `genutil.ValidateAccountInGenesis()`, `genutil.GenAppStateFromConfig()`.
- **EVM integration**: `ethcrypto.PubkeyToAddress()` for EVM address derivation.

Note: seictl carries `replace` directives from sei-chain's go.mod because Go ignores replace directives from transitive dependencies.

## Relationship to the Kubernetes Operator

seictl and the sei-k8s-controller are **complementary, not overlapping**:

- **The controller** (sei-k8s-controller) runs as a Kubernetes operator. It creates StatefulSets, Services, and PVCs for Sei nodes. It orchestrates the bootstrap sequence by submitting tasks to each node's sidecar.

- **seictl** runs as a sidecar container in each Sei node's pod. It executes the actual work (downloading snapshots, generating keys, writing config files, etc.) on the node's local filesystem.

- **Communication**: The controller talks to seictl via the sidecar HTTP API. The `sidecar/client/` package in seictl provides the typed Go client that the controller imports (aliased as `sidecar` by convention). The client constructs URLs using Kubernetes headless-service DNS: `http://{name}-0.{name}.{namespace}.svc.cluster.local:7777`.

- **Task ID determinism**: The controller can provide deterministic UUIDs for tasks, enabling idempotent submission. If a task with the same ID already exists, the sidecar returns the existing ID without re-submitting.

The bootstrap sequence driven by the controller is:
1. `generate-identity` -- create node keys
2. `generate-gentx` -- create genesis transaction (for genesis ceremonies)
3. `upload-genesis-artifacts` -- upload to S3 (one node per group)
4. `assemble-and-upload-genesis` -- collect all gentxs, produce final genesis (coordinator node)
5. `configure-genesis` -- download final genesis (or use embedded for well-known chains)
6. `set-genesis-peers` -- configure persistent peers from genesis ceremony
7. `config-apply` -- generate full config from intent
8. `discover-peers` -- resolve peers from EC2 tags or static list
9. `config-patch` -- apply any additional config overrides
10. `snapshot-restore` -- download and extract snapshot
11. `configure-state-sync` -- set trust point for state sync
12. `mark-ready` -- signal bootstrap complete (healthz goes 200, seid container can start)
13. `snapshot-upload` (scheduled) -- periodic snapshot archival
14. `result-export` (scheduled) -- periodic block result export

## Infrastructure Target

seictl targets **Kubernetes** as its primary deployment platform:
- Runs as a sidecar container in pods managed by StatefulSets.
- Uses Kubernetes headless-service DNS for pod-to-pod communication.
- Uses shared process namespace (`shareProcessNamespace: true`) for process signaling.
- Uses AWS S3 for snapshot storage, genesis artifact exchange, and result export.
- Uses AWS EC2 API for peer discovery (legacy path; controller-driven resolution is the future).
- Built as `CGO_ENABLED=0` static binary on distroless base image.

The CLI commands (`config patch`, `genesis patch`, `patch`, `await`) are also usable on bare metal or VMs for manual node operation, independent of Kubernetes.

## Key Dependencies

| Dependency | Purpose |
|---|---|
| `urfave/cli/v3` | CLI framework |
| `pelletier/go-toml/v2` | TOML parsing and writing |
| `sei-protocol/sei-config` | Config intent resolution, validation, embedded genesis |
| `sei-protocol/sei-chain` | CometBFT/Cosmos SDK types, key generation, genesis manipulation |
| `sei-protocol/seilog` | Structured logging |
| `aws/aws-sdk-go-v2` | S3 uploads/downloads, EC2 peer discovery |
| `oapi-codegen/runtime` | OpenAPI client generation |
| `robfig/cron/v3` | Cron expression parsing for scheduled tasks |
| `google/uuid` | Task ID generation |
| `ethereum/go-ethereum` | EVM address derivation (fork: sei-protocol/go-ethereum) |

## Build and Distribution

- **Binary**: `make build` produces `./build/seictl`.
- **Docker**: Multi-stage build; final image is `gcr.io/distroless/static-debian12` with `/usr/bin/seictl`.
- **Release**: GoReleaser produces native binaries for 7 platform/arch combinations + Docker image via GHCR.
- **Client generation**: `make generate` runs oapi-codegen from `sidecar/api/openapi.yaml` to produce `sidecar/client/sidecar.gen.go`.

## Design Notes

- All long-running tasks use **marker files** (`.sei-sidecar-*-done`) for idempotency. Once a task completes, it is skipped on re-invocation.
- S3 operations use the **transfer manager** for parallel byte-range downloads and multipart uploads.
- Snapshot upload uses `io.Pipe` to stream archives directly to S3 without buffering the full archive in memory.
- The genesis ceremony is fully programmatic -- no shell calls to `seid`. The sidecar calls the same Go SDK functions that `seid init/gentx/collect-gentxs` use internally.
- The `config-reload` task validates hot-reloadability against sei-config's field registry before applying changes.
- Config keys in seid use **hyphens** (e.g., `persistent-peers`, `trust-height`), matching CometBFT convention.
