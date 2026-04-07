---
topic: "seienv: legacy CLI for Sei node operations on EC2"
sources:
  - repo: slanders (sei-protocol/slanders)
    version: main branch
    files:
      - seienv/README.md
      - seienv/cheatsheet.md
      - seienv/cmd/upgrade/upgrade.go
      - seienv/cmd/upgrade/scripts/upgrade.sh
      - seienv/cmd/replace/replace.go
      - seienv/cmd/propose/propose.go
      - seienv/cmd/propose/scripts/propose.sh
      - seienv/common/remotescript.go
verified: 2026-04-07
confidence: high
---

## Overview

`seienv` is a Go CLI tool that operates on Sei nodes deployed as EC2 instances. It uses AWS EC2 tags to discover nodes by chain ID and component type, then SSHes into them to execute scripts. It is the legacy operational tool that the k8s controller replaces.

## Architecture

```
seienv (local laptop) → AWS EC2 DescribeInstances (by tags) → SSH into nodes → execute bash scripts
```

- **Discovery:** AWS EC2 tags — `ChainIdentifier` for chain, `Component` for role (validators, webapp, state-syncer, etc.)
- **Execution:** SCP a bash script to `/tmp/seienv-script.sh`, then `sudo /tmp/seienv-script.sh <args>` via SSH
- **Auth:** AWS SSO profile + SSH key (`~/.ssh/sei.pem`)
- **Concurrency:** Configurable worker threads (default 50) via `errgroup`
- **Node selection:** `--chain-id`, `--component` (default: validators), `--id`, `--ip`, `--name`

## Upgrade Process

The upgrade flow is the most important operation for the controller to replace.

### Step 1: Propose Upgrade (`seienv propose-upgrade v6.4.0 --seconds 500`)

1. Queries a node's RPC for the current block height
2. Calculates `futureHeight = currentHeight + (seconds * 3)` (assumes ~3 blocks/sec)
3. SSHes into ONE node and runs:
   ```bash
   seid tx gov submit-proposal software-upgrade "$version" \
     --title "$version" --from "$from" --fees 20000usei \
     -b block -y --upgrade-height=$height \
     --description "changelog URL" \
     --deposit "${deposit}${denom}" --chain-id "$chain_id"
   ```
4. Outputs the proposal ID

### Step 2: Vote (`seienv vote 5 yes --fees 20sei --from node_admin`)

Runs `seid tx gov vote` on all validators.

### Step 3: At Upgrade Height

The Cosmos SDK upgrade module's `BeginBlocker` detects the upgrade height, writes `upgrade-info.json`, and panics the node. All nodes halt.

### Step 4: Upgrade Binary (`seienv upgrade v6.4.0 --restart`)

SSHes into ALL targeted nodes and runs `upgrade.sh`:
```bash
rm -rf sei-chain
git clone $repo
cd sei-chain
git checkout "$version"
make install        # builds seid binary
service seid restart
```

**Critical: this clones and builds from source on every node.** There is no pre-built binary distribution. The `--restart` flag restarts seid after building; `--reset` wipes data and restarts.

### Replace Sub-Library (`seienv replace sei-tendermint <commit> --restart`)

Similar flow but only replaces one Go module:
```bash
cd sei-chain
go get github.com/sei-protocol/$library@$version
make install
service seid restart
```

Supports: sei-tendermint, sei-cosmos, sei-wasmd, go-ethereum.

## Other Key Operations

| Command | What It Does |
|---------|-------------|
| `seienv ls` | Lists EC2 IPs by component/chain tags |
| `seienv ls --versions` | SSHes into each node, runs `seid version` |
| `seienv ssh` | Opens interactive SSH to first matching node |
| `seienv logs` | `journalctl -fu seid` on a node |
| `seienv service seid restart` | `service seid restart` on all matching nodes |
| `seienv reset` | Stops seid, wipes `~/.sei/data` and `~/.sei/wasm`, restarts |
| `seienv exec --command "ps -eaf"` | Runs arbitrary command on all nodes |
| `seienv exec --script script.sh` | Copies and executes a local script on all nodes |
| `seienv cp file.txt /tmp/file.txt` | SCP a file to all nodes |
| `seienv set_config` | Modifies config on nodes |
| `seienv trace start/stop` | Installs docker + Jaeger for tracing |

## What seienv Does NOT Do

- No health checking or readiness gating
- No rolling upgrades — all nodes upgrade simultaneously
- No blue-green deployment
- No automatic rollback on failure
- No state sync or snapshot management
- No peer discovery automation (peers are manually configured or rely on existing address books)
- No config validation before applying
- No idempotency — re-running an upgrade rebuilds everything from scratch

## Protected Environments

`reset` and other destructive commands are blocked for arctic, atlantic, and pacific chains.

## Key Takeaways

- **seienv is SSH + bash scripts orchestrated from a laptop.** The k8s controller replaces this with declarative CRDs, sidecar tasks, and Kubernetes-native lifecycle management.
- **Upgrades are build-from-source on every node.** The controller uses pre-built container images.
- **No safety rails.** seienv has no health checks, no readiness gates, no rollback. The controller adds `/lag_status` probes, blue-green deployments, and plan-based orchestration.
- **Peer configuration is manual.** seienv doesn't manage persistent_peers. The controller resolves peers from labels, EC2 tags, and DNS.
- **The upgrade proposal flow is the same regardless of tooling** — it's a governance tx. The controller needs to handle the binary-swap-and-restart part, not the proposal part.
