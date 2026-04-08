---
title: "Cosmovisor & Cosmos SDK Upgrade Module: Complete Mechanics"
author: "RALPHY (Tide Research Agent)"
date: "2026-04-07"
sources:
  sei-cosmos:
    version: "v0.3.66"
    path: "~/go/pkg/mod/github.com/sei-protocol/sei-cosmos@v0.3.66/"
    files_read:
      - "x/upgrade/abci.go"
      - "x/upgrade/module.go"
      - "x/upgrade/handler.go"
      - "x/upgrade/doc.go"
      - "x/upgrade/keeper/keeper.go"
      - "x/upgrade/keeper/grpc_query.go"
      - "x/upgrade/types/plan.go"
      - "x/upgrade/types/keys.go"
      - "x/upgrade/types/storeloader.go"
      - "x/upgrade/types/handler.go"
      - "x/upgrade/types/upgrade.pb.go"
      - "baseapp/abci.go"
      - "server/start.go"
      - "server/in_place.go"
      - "server/config/config.go"
  sei-tendermint:
    version: "v0.6.4"
    path: "~/go/pkg/mod/github.com/sei-protocol/sei-tendermint@v0.6.4/"
    files_read:
      - "node/node.go"
      - "config/config.go"
      - "internal/blocksync/reactor.go"
  cosmovisor:
    version: "cosmos-sdk v0.47.5 (upstream reference)"
    source: "github.com/cosmos/cosmos-sdk/tools/cosmovisor/"
    files_read:
      - "args.go"
      - "process.go"
      - "upgrade.go"
      - "scanner.go"
---

# Cosmovisor & Cosmos SDK Upgrade Module: Complete Mechanics

## Part 1: On-Chain Upgrade Module (x/upgrade)

### 1.1 The Plan Struct

The upgrade plan is a protobuf message stored on-chain:

```go
// From x/upgrade/types/upgrade.pb.go
type Plan struct {
    Name                string     // Unique upgrade name, e.g. "v5.0.0"
    Time                time.Time  // DEPRECATED - time-based upgrades removed
    Height              int64      // Block height at which upgrade executes
    Info                string     // Arbitrary JSON - used for cosmovisor auto-download
    UpgradedClientState *types.Any // DEPRECATED - IBC upgrade moved to IBC module
}
```

The `Info` field can contain an `UpgradeDetails` JSON:
```go
type UpgradeDetails struct {
    UpgradeType string `json:"upgradeType"` // "minor" for minor releases
}
```

If `UpgradeType` is `"minor"` (case-insensitive), the upgrade has special handling (see below).

### 1.2 KV Store Layout

All upgrade state lives under the `"upgrade"` store key:

| Prefix Byte | Key Pattern | Value | Purpose |
|---|---|---|---|
| `0x0` | `[]byte{0x0}` (PlanKey) | Protobuf-marshaled `Plan` | Currently scheduled upgrade plan |
| `0x1` | `0x1 + name` (DoneByte) | uint64 big-endian height | Marks upgrade as completed at height |
| `0x2` | `0x2 + moduleName` (VersionMapByte) | uint64 big-endian version | Per-module consensus version |
| `0x3` | `[]byte{0x3}` (ProtocolVersionByte) | uint64 big-endian version | Protocol version counter |

### 1.3 ScheduleUpgrade: How Plans Are Stored

When governance passes a `SoftwareUpgradeProposal`, the handler calls `keeper.ScheduleUpgrade`:

```go
func (k Keeper) ScheduleUpgrade(ctx sdk.Context, plan types.Plan) error {
    // 1. Basic validation (name non-empty, height > 0, no time-based)
    if err := plan.ValidateBasic(); err != nil { return err }

    // 2. Cannot schedule in the past (but CAN schedule for current block - emergency hard forks)
    if plan.Height < ctx.BlockHeight() { return error }

    // 3. Cannot reuse a completed upgrade name
    if k.GetDoneHeight(ctx, plan.Name) != 0 { return error }

    // 4. Clear any old IBC state from previous plan
    oldPlan, found := k.GetUpgradePlan(ctx)
    if found { k.ClearIBCState(ctx, oldPlan.Height) }

    // 5. Marshal and store under PlanKey (0x0)
    bz := k.cdc.MustMarshal(&plan)
    store.Set(types.PlanKey(), bz)
}
```

Key detail: scheduling for `ctx.BlockHeight()` (current height) is explicitly allowed. This enables emergency hard fork recovery where a proposal can schedule an upgrade in the same block's BeginBlock.

### 1.4 BeginBlocker: Line-by-Line Trace

This is the core of the upgrade module. It runs **before all other modules' BeginBlockers** (contractual requirement noted in module.go).

```go
func BeginBlocker(k keeper.Keeper, ctx sdk.Context, _ abci.RequestBeginBlock) {
```

**Phase 0: Tracing guard**
```go
    if ctx.IsTracing() { return }
```
If the context is in tracing mode, skip entirely.

**Phase 1: Downgrade verification (runs ONCE per binary lifetime)**
```go
    if !k.DowngradeVerified() {
        k.SetDowngradeVerified(true)
        lastAppliedPlan, _ := k.GetLastCompletedUpgrade(ctx)
```
This is an in-memory boolean on the Keeper (not persisted to state). It checks exactly once after binary startup whether we're running the correct binary version.

The check fires in three cases:
1. No plan is scheduled (`!planFound`)
2. Plan exists but is not yet due (`!plan.ShouldExecute(ctx)`)
3. Plan is due but this height is in skip-upgrades (`plan.ShouldExecute(ctx) && k.IsSkipHeight(...)`)

In all three cases, if there was a previously applied upgrade and the current binary has NO handler for it:
```go
        if lastAppliedPlan != "" && !k.HasHandler(lastAppliedPlan) {
            panic(fmt.Sprintf("Wrong app version %d, upgrade handler is missing for %s upgrade plan",
                ctx.ConsensusParams().Version.AppVersion, lastAppliedPlan))
        }
```
**This means: if you restart with the OLD binary after an upgrade has been applied, the node panics immediately on the first block.** The old binary won't have a handler for the upgrade that was already completed.

**Phase 2: No plan? Return early.**
```go
    if !planFound { return }
```

**Phase 3: Telemetry emission** (every block while a plan exists)

**Phase 4: Plan is due for execution (`plan.ShouldExecute(ctx)`)**

`ShouldExecute` returns true when `plan.Height <= ctx.BlockHeight()`. This means the plan triggers at the specified height AND at every subsequent height if not yet applied.

```go
    if plan.ShouldExecute(ctx) {
        // 4a: Skip if height is in --unsafe-skip-upgrades
        if k.IsSkipHeight(ctx.BlockHeight()) {
            skipUpgrade(k, ctx, plan)  // logs + ClearUpgradePlan
            return
        }

        // 4b: No handler registered = OLD binary = halt needed
        if !k.HasHandler(plan.Name) {
            panicUpgradeNeeded(k, ctx, plan)  // writes upgrade-info.json then panics
        }

        // 4c: Handler exists = NEW binary = apply the upgrade
        applyUpgrade(k, ctx, plan)
        return
    }
```

**Phase 5: Plan exists but is NOT yet due**

```go
    details, err := plan.UpgradeDetails()

    // 5a: Minor release - allowed to run early
    if details.IsMinorRelease() {
        if !k.HasHandler(plan.Name) && !k.IsSkipHeight(plan.Height) {
            if ctx.BlockHeight()%100 == 0 {
                ctx.Logger().Info(BuildUpgradeScheduledMsg(plan))
            }
        }
        return
    }

    // 5b: Major release with handler = binary updated too early = PANIC
    if k.HasHandler(plan.Name) {
        panic(fmt.Sprintf("BINARY UPDATED BEFORE TRIGGER! UPGRADE \"%s\" - in binary but not executed on chain", plan.Name))
    }
```

**CRITICAL: For major upgrades, if the NEW binary is running BEFORE the upgrade height, it panics.** This prevents nodes from running new code before the network-coordinated switch point.

### 1.5 The Panic: Exact Messages and Timing

There are **four distinct panic messages** in the upgrade module:

| Panic Message | When | Binary |
|---|---|---|
| `UPGRADE "<name>" NEEDED at height: <N>: <info>` | Upgrade height reached, no handler | OLD binary |
| `Wrong app version %d, upgrade handler is missing for %s upgrade plan` | After restart, last completed upgrade has no handler | OLD binary (post-upgrade restart) |
| `BINARY UPDATED BEFORE TRIGGER! UPGRADE "<name>" - in binary but not executed on chain` | Before upgrade height, handler exists for pending major upgrade | NEW binary (started too early) |
| `unable to write upgrade info to filesystem: <err>` | Failed to write upgrade-info.json | OLD binary (filesystem error) |

**When does the panic happen relative to block commit?**

The panic occurs in **BeginBlock**, which runs BEFORE any transactions are executed and BEFORE Commit. The block at the upgrade height is never committed by the old binary. The Cosmos SDK doc.go confirms: "the upgrade keeper simply panics which prevents the ABCI state machine from proceeding but doesn't actually exit the process."

The panic bubbles up through the ABCI BeginBlock call. Tendermint catches panics differently depending on the ABCI transport, but the effect is the same: the state machine halts, no further blocks are processed.

### 1.6 upgrade-info.json

Written by `panicUpgradeNeeded` just BEFORE the panic:

```go
func panicUpgradeNeeded(k keeper.Keeper, ctx sdk.Context, plan types.Plan) {
    err := k.DumpUpgradeInfoWithInfoToDisk(ctx.BlockHeight(), plan.Name, plan.Info)
    if err != nil {
        panic(fmt.Errorf("unable to write upgrade info to filesystem: %s", err.Error()))
    }
    panic(BuildUpgradeNeededMsg(plan))
}
```

**File location:** `{DAEMON_HOME}/data/upgrade-info.json`

```go
func (k Keeper) GetUpgradeInfoPath() (string, error) {
    upgradeInfoFileDir := path.Join(k.getHomeDir(), "data")
    return filepath.Join(upgradeInfoFileDir, "upgrade-info.json"), nil
}
```

**File contents:**
```json
{
    "name": "v5.0.0",
    "height": 12345678,
    "info": "{\"binaries\":{\"linux/amd64\":\"https://...\"}}"
}
```

The `info` field contains `Plan.Info` verbatim - this is what Cosmovisor uses for auto-download.

**When is it written?** Only when the old binary panics at the upgrade height. It is written BEFORE the panic.

**When is it read?** At app startup, the new binary calls `ReadUpgradeInfoFromDisk()` to determine which store migrations to apply:
```go
upgradeInfo, err := app.UpgradeKeeper.ReadUpgradeInfoFromDisk()
if upgradeInfo.Name == "my-fancy-upgrade" && !app.UpgradeKeeper.IsSkipHeight(upgradeInfo.Height) {
    app.SetStoreLoader(upgrade.UpgradeStoreLoader(upgradeInfo.Height, &storeUpgrades))
}
```

### 1.7 ApplyUpgrade: What the New Binary Does

```go
func (k Keeper) ApplyUpgrade(ctx sdk.Context, plan types.Plan) {
    // 1. Run the upgrade handler (migration logic)
    handler := k.upgradeHandlers[plan.Name]
    updatedVM, err := handler(ctx, plan, k.GetModuleVersionMap(ctx))

    // 2. Store updated module version map
    k.SetModuleVersionMap(ctx, updatedVM)

    // 3. Increment protocol version
    nextProtocolVersion := k.getProtocolVersion(ctx) + 1
    k.setProtocolVersion(ctx, nextProtocolVersion)
    if k.versionSetter != nil {
        k.versionSetter.SetProtocolVersion(nextProtocolVersion)
    }

    // 4. Clear IBC state and the plan itself
    k.ClearIBCState(ctx, plan.Height)
    k.ClearUpgradePlan(ctx)

    // 5. Mark this upgrade as done (stores height under DoneByte prefix)
    k.SetDone(ctx, plan.Name)
}
```

After `ApplyUpgrade`, the plan is deleted from state, the upgrade name is recorded in the "done" map with the current block height, and the protocol version is incremented. The block then proceeds normally (DeliverTx, EndBlock, Commit).

### 1.8 DoneHeight: Marking Upgrades Complete

```go
func (k Keeper) SetDone(ctx sdk.Context, name string) {
    store := prefix.NewStore(ctx.KVStore(k.storeKey), []byte{types.DoneByte})
    bz := make([]byte, 8)
    binary.BigEndian.PutUint64(bz, uint64(ctx.BlockHeight()))
    store.Set([]byte(name), bz)
}
```

The "done" record persists forever. It prevents:
1. Re-scheduling an upgrade with the same name (`ScheduleUpgrade` checks `GetDoneHeight`)
2. Allows `GetLastCompletedUpgrade` to find the most recently applied upgrade for downgrade verification

### 1.9 --unsafe-skip-upgrades

The flag is defined in `server/start.go`:
```go
cmd.Flags().IntSlice(FlagUnsafeSkipUpgrades, []int{},
    "Skip a set of upgrade heights to continue the old binary")
```

These heights are passed to the Keeper constructor as `skipUpgradeHeights map[int64]bool`. In BeginBlocker, when an upgrade is due:
```go
if k.IsSkipHeight(ctx.BlockHeight()) {
    skipUpgrade(k, ctx, plan)  // ClearUpgradePlan, log message
    return
}
```

This clears the plan without applying any migration, allowing the old binary to continue. It requires >2/3 of validators to run with this flag for the chain to proceed.

### 1.10 What Happens If Old Binary Restarts After Upgrade Height

**Scenario:** Upgrade applied at height 1000 by new binary. Operator mistakenly restarts with old binary.

The downgrade verification check fires:
```go
lastAppliedPlan, _ := k.GetLastCompletedUpgrade(ctx)
// lastAppliedPlan = "v5.0.0", height = 1000
// Old binary has no handler for "v5.0.0"
if lastAppliedPlan != "" && !k.HasHandler(lastAppliedPlan) {
    panic("Wrong app version ..., upgrade handler is missing for v5.0.0 upgrade plan")
}
```

**The old binary panics immediately on the first BeginBlock.** It never processes any blocks. This is a safety mechanism.

### 1.11 Minor vs Major Upgrades (Sei-Specific)

Sei-cosmos adds a distinction not in upstream:

- **Major upgrades**: MUST execute exactly at the specified height. Running the new binary before that height causes a panic. Running the old binary at that height causes a panic.
- **Minor upgrades**: `Plan.Info` contains `{"upgradeType":"minor"}`. These are allowed to run before the scheduled height. The BeginBlocker simply returns without panicking if the handler is missing (it just logs every 100 blocks).

This allows rolling minor updates without coordinated halt.

---

## Part 2: halt-height (Separate from x/upgrade)

The `--halt-height` flag is a BaseApp feature, independent of the upgrade module:

```go
// In baseapp/abci.go Commit():
switch {
case app.haltHeight > 0 && uint64(header.Height) >= app.haltHeight:
    halt = true
case app.haltTime > 0 && header.Time.Unix() >= int64(app.haltTime):
    halt = true
}

if halt {
    app.halt()
}
```

**Key differences from x/upgrade panic:**
1. halt-height triggers AFTER Commit (block IS committed), not in BeginBlock
2. It sends SIGINT/SIGTERM to self (graceful shutdown), not a panic
3. It's a clean exit, not a crash loop
4. No upgrade-info.json is written

```go
func (app *BaseApp) halt() {
    app.logger.Info("halting node per configuration", "height", app.haltHeight, "time", app.haltTime)
    p, _ := os.FindProcess(os.Getpid())
    sigIntErr := p.Signal(syscall.SIGINT)
    sigTermErr := p.Signal(syscall.SIGTERM)
    if sigIntErr == nil || sigTermErr == nil { return }
    os.Exit(0)
}
```

---

## Part 3: Cosmovisor (Reference Implementation)

Cosmovisor is a process supervisor that wraps the daemon binary (`seid`). It is NOT bundled in sei-cosmos; it lives in the upstream `cosmos-sdk/tools/cosmovisor/` package.

### 3.1 Directory Structure

```
$DAEMON_HOME/
  cosmovisor/
    genesis/
      bin/
        seid              # Initial binary
    upgrades/
      v5.0.0/
        bin/
          seid            # Binary for v5.0.0 upgrade
      v6.0.0/
        bin/
          seid            # Binary for v6.0.0 upgrade
    current -> genesis/   # Symlink to active version dir
  data/
    upgrade-info.json     # Written by x/upgrade on panic
```

### 3.2 Configuration (Environment Variables)

| Variable | Default | Purpose |
|---|---|---|
| `DAEMON_HOME` | required | Node home directory |
| `DAEMON_NAME` | required | Binary name (e.g., `seid`) |
| `DAEMON_ALLOW_DOWNLOAD_BINARIES` | `false` | Auto-download new binaries from `Plan.Info` |
| `DAEMON_RESTART_AFTER_UPGRADE` | `true` | Restart after upgrade binary swap |
| `DAEMON_RESTART_DELAY` | `0` | Delay before restart after upgrade |
| `DAEMON_POLL_INTERVAL` | `300ms` | How often to poll upgrade-info.json |
| `UNSAFE_SKIP_BACKUP` | `false` | Skip data backup before upgrade |
| `DAEMON_DATA_BACKUP_DIR` | `$DAEMON_HOME` | Where to put backups |
| `DAEMON_PREUPGRADE_MAX_RETRIES` | `0` | Max retries for pre-upgrade command |

### 3.3 Upgrade Detection: File Watcher (Not Log Parsing)

Cosmovisor does **NOT** parse logs. It uses a **file-based polling mechanism** watching `$DAEMON_HOME/data/upgrade-info.json`.

The `fileWatcher` struct polls the filesystem at `DAEMON_POLL_INTERVAL` (default 300ms):

```go
func (fw *fileWatcher) CheckUpdate(currentUpgrade upgradetypes.Plan) bool {
    stat, err := os.Stat(fw.filename)
    if err != nil { return false }  // file doesn't exist yet

    if !stat.ModTime().After(fw.lastModTime) { return false }

    info, err := parseUpgradeInfoFile(fw.filename)
    // ...

    // On first check after restart (daemon restarted):
    if !fw.initialized {
        fw.initialized = true
        fw.currentInfo = info
        // If running upgrade name != file's upgrade name, we need to update
        if !strings.EqualFold(currentUpgrade.Name, fw.currentInfo.Name) {
            fw.needsUpdate = true
            return true
        }
    }

    // Otherwise: new upgrade detected if height increased
    if info.Height > fw.currentInfo.Height {
        fw.needsUpdate = true
        return true
    }
    return false
}
```

**Important edge case:** The app's x/upgrade panic may kill the process BEFORE the file watcher detects the new upgrade-info.json. Cosmovisor handles this in `WaitForUpgradeOrExit`:

```go
case err := <-cmdDone:
    l.fw.Stop()
    if err == nil { return false, nil }
    // The app died (panic) - recheck the file one more time
    if !l.fw.CheckUpdate(currentUpgrade) {
        return false, err  // genuine crash, not upgrade
    }
    // File was updated -> treat as upgrade
```

### 3.4 Binary Swap: Symlinks

Cosmovisor uses **symlinks**, not copies:

```go
func (cfg *Config) SetCurrentUpgrade(u upgradetypes.Plan) error {
    bin := cfg.UpgradeBin(u.Name)
    if err := EnsureBinary(bin); err != nil { return err }

    link := filepath.Join(cfg.Root(), currentLink)  // cosmovisor/current
    safeName := url.PathEscape(u.Name)
    upgrade := filepath.Join(cfg.Root(), upgradesDir, safeName)

    // Remove old symlink
    if _, err := os.Stat(link); err == nil {
        os.Remove(link)
    }

    // Create new: current -> upgrades/v5.0.0/
    os.Symlink(upgrade, link)
    cfg.currentUpgrade = u

    // Also write upgrade-info.json into the upgrade dir
    // (separate from the data/upgrade-info.json written by x/upgrade)
    f, _ := os.Create(filepath.Join(upgrade, "upgrade-info.json"))
    json.NewEncoder(f).Encode(u)
}
```

`CurrentBin()` resolves the symlink:
```go
func (cfg *Config) CurrentBin() (string, error) {
    cur := filepath.Join(cfg.Root(), currentLink)
    dest, _ := os.Readlink(cur)
    return filepath.Join(dest, "bin", cfg.Name), nil
}
```

### 3.5 Restart Flow: exec.Command (Fork, Not Exec)

Cosmovisor forks a new child process, it does NOT use `syscall.Exec`:

```go
func (l Launcher) Run(args []string, stdout, stderr io.Writer) (bool, error) {
    bin, _ := l.cfg.CurrentBin()
    cmd := exec.Command(bin, args...)
    cmd.Stdout = stdout
    cmd.Stderr = stderr
    cmd.Start()

    // Forward signals to child
    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGQUIT, syscall.SIGTERM)
    go func() {
        sig := <-sigs
        cmd.Process.Signal(sig)
    }()

    // Wait for upgrade detection or process exit
    needsUpdate, err := l.WaitForUpgradeOrExit(cmd)
    if !needsUpdate { return false, err }

    // Upgrade detected:
    l.cfg.WaitRestartDelay()
    l.doBackup()                              // Copy data/ dir
    UpgradeBinary(l.logger, l.cfg, info)      // Swap symlink
    l.doPreUpgrade()                          // Run `seid pre-upgrade`
    return true, nil  // Caller loops and calls Run() again
}
```

The caller (cosmovisor's main) loops on `Run()` returning `true`:
```
while Run() returns (true, nil):
    Run() again with new binary
```

### 3.6 Auto-Download (DAEMON_ALLOW_DOWNLOAD_BINARIES)

When enabled, Cosmovisor downloads the binary from URLs in `Plan.Info`:

```go
func UpgradeBinary(logger, cfg, info) error {
    // 1. Check if binary already exists at upgrades/{name}/bin/{daemon}
    err := EnsureBinary(cfg.UpgradeBin(info.Name))
    if err == nil {
        return cfg.SetCurrentUpgrade(info)  // Already have it
    }

    // 2. If auto-download disabled, fail
    if !cfg.AllowDownloadBinaries {
        return error
    }

    // 3. Download from Plan.Info
    DownloadBinary(cfg, info)

    // 4. Verify and set as current
    EnsureBinary(cfg.UpgradeBin(info.Name))
    return cfg.SetCurrentUpgrade(info)
}
```

The `Plan.Info` JSON format for auto-download:
```json
{
    "binaries": {
        "linux/amd64": "https://example.com/seid-v5.0.0-linux-amd64",
        "darwin/arm64": "https://example.com/seid-v5.0.0-darwin-arm64",
        "any": "https://example.com/seid-v5.0.0-universal"
    }
}
```

Downloads use `go-getter` which supports `https://`, `s3://`, `gcs://`, and zip/tar extraction.

### 3.7 Pre-Upgrade Command

After binary swap but before restart, Cosmovisor runs `{new_binary} pre-upgrade`:

```go
func (l *Launcher) doPreUpgrade() error {
    for counter := 0; counter <= l.cfg.PreupgradeMaxRetries; counter++ {
        err := l.executePreUpgradeCmd()
        if err == nil { return nil }  // Success

        switch err.(*exec.ExitError).ProcessState.ExitCode() {
        case 1:   return nil   // Command doesn't exist, OK
        case 30:  return err   // Fatal failure
        case 31:  continue     // Retry
        }
    }
}
```

### 3.8 Cosmovisor Failure Modes

| Failure | Behavior |
|---|---|
| New binary not found & auto-download disabled | Cosmovisor exits with error. Node stays down. |
| New binary download fails | Cosmovisor exits with error. Node stays down. |
| New binary fails to start (crash) | NOT handled by Cosmovisor itself. Process exits. External supervisor (systemd) must restart, creating crash loop. |
| Backup fails (disk full) | Cosmovisor exits with error before swap. Node stays down with old binary still "current". |
| Pre-upgrade command fails (exit 30) | Cosmovisor exits with error. Binary was already swapped. Next restart uses new binary. |
| upgrade-info.json malformed | `Fatal()` log - cosmovisor exits |
| Symlink creation fails | Error return, cosmovisor exits |

**Critical gap:** Cosmovisor has NO rollback mechanism. If the new binary is set as "current" but then fails to start, the symlink stays pointing to the new binary. Manual intervention is required.

---

## Part 4: Sei-Tendermint Self-Remediation

Sei-tendermint adds a `SelfRemediationConfig` that is distinct from both x/upgrade and Cosmovisor:

```go
type SelfRemediationConfig struct {
    P2pNoPeersRestarWindowSeconds        uint64  // Restart if no peers for N seconds
    StatesyncNoPeersRestartWindowSeconds uint64  // Restart if no statesync peers for N seconds
    BlocksBehindThreshold                uint64  // Restart if N blocks behind peers
    BlocksBehindCheckIntervalSeconds     uint64  // How often to check (default: 60s)
    RestartCooldownSeconds               uint64  // Min time between restarts (default: 600s)
}
```

Self-remediation sends restart signals via a channel, triggering a node-level restart (not a binary swap). This is for liveness, not upgrades.

---

## Part 5: Controller Implications

### 5.1 What the K8s Controller Can Do BETTER Than Cosmovisor

| Capability | Cosmovisor | K8s Controller |
|---|---|---|
| **Fleet visibility** | None (single node) | Full fleet status, can orchestrate rolling upgrades |
| **Pre-staging binaries** | Must pre-place or auto-download | Update container image tag, pull happens automatically |
| **Rollback** | None | Change image tag back, StatefulSet rollback |
| **Monitoring upgrade progress** | Process exit code only | Watch pod status, health endpoints, block height across fleet |
| **Upgrade ordering** | None (all nodes halt simultaneously) | Can upgrade sentries first, then validators |
| **Backup coordination** | Local disk copy only | Can trigger snapshots, S3 backups before upgrade |
| **Binary validation** | `EnsureBinary()` checks execute bit | Container image validation, init container checks |
| **Crash loop detection** | None (relies on systemd) | Pod restart count, backoff detection, automatic intervention |
| **Multi-version fleet** | Not possible | Can run mixed versions during rolling update |

### 5.2 Minimum Viable Upgrade Automation

The controller needs to handle this timeline:

```
1. Governance proposal passes → Plan stored on-chain
2. Blocks proceed normally until plan.Height
3. At plan.Height, BeginBlocker panics → node crash loops
4. WINDOW: Node is down, waiting for new binary
5. New image applied → pod restarts with new binary
6. New binary's BeginBlocker finds handler → ApplyUpgrade()
7. Block commits, chain proceeds
```

**Minimum viable steps:**

1. **Detect plan**: Query `x/upgrade` CurrentPlan gRPC endpoint (`/cosmos.upgrade.v1beta1.Query/CurrentPlan`). This returns the pending plan with name, height, and info.

2. **Wait for halt**: Monitor node status. When block height reaches `plan.Height`, the pod will crash (exit code from panic). The controller sees repeated container restarts.

3. **Swap image**: Update the StatefulSet's container image to the new version. This is the equivalent of Cosmovisor's symlink swap.

4. **Restart**: Kubernetes handles this automatically when the StatefulSet spec changes.

### 5.3 Handling the Crash-Loop Window

Between "upgrade height reached" and "new image applied", the node WILL crash-loop. Each restart:
1. Binary starts
2. Replays to upgrade height
3. BeginBlocker fires
4. No handler found → writes upgrade-info.json → panics
5. Kubernetes restarts the container
6. Go to step 1

**Controller strategies:**

**Option A: Proactive image swap (preferred)**
- Controller detects pending plan via gRPC query
- When current height approaches plan.Height (e.g., within N blocks), proactively update the image
- The new binary starts, has the handler, and will apply the upgrade at the correct height
- **Risk**: If the new binary is deployed before plan.Height for a MAJOR upgrade, the new binary panics with "BINARY UPDATED BEFORE TRIGGER". The new binary must NOT start processing blocks before the upgrade height.
- **Mitigation**: The new binary WILL have the handler but the code explicitly checks: if the plan exists, is NOT yet due, is NOT minor, and a handler IS registered → panic. So early deployment of the new binary also causes crash-looping until the upgrade height is reached. This is actually safe but noisy.

**Option B: Reactive image swap (simpler)**
- Let the old binary hit the upgrade height and start crash-looping
- Controller detects the crash loop (or watches block height reach plan.Height)
- Swap the image
- Crash-loop stops once the new binary comes up
- **Downside**: Window of crash-looping, which wastes time and creates noise

**Option C: halt-height coordination (cleanest)**
- Controller sets `--halt-height` to `plan.Height - 1` in the node args
- Node halts cleanly BEFORE the upgrade (no panic, graceful shutdown)
- Controller swaps the image
- New binary starts at the upgrade height and applies the upgrade
- **Note**: halt-height halts AFTER committing that block, so setting it to `plan.Height - 1` means the last committed block is `plan.Height - 1`, and the new binary starts processing `plan.Height`.
- **This is the cleanest approach** but requires the controller to detect plans early enough to inject halt-height.

### 5.4 Querying Upgrade State

The controller can use these gRPC endpoints:

```
// Get pending plan
/cosmos.upgrade.v1beta1.Query/CurrentPlan
→ { plan: { name: "v5", height: 12345, info: "..." } }

// Check if upgrade was applied
/cosmos.upgrade.v1beta1.Query/AppliedPlan
→ { height: 12345 }  // non-zero = applied

// Get module versions
/cosmos.upgrade.v1beta1.Query/ModuleVersions
→ [{ name: "bank", version: 4 }, ...]
```

Or read `upgrade-info.json` from the PVC:
```
{DAEMON_HOME}/data/upgrade-info.json
→ { "name": "v5", "height": 12345, "info": "..." }
```

### 5.5 Summary of Upgrade Flow State Machine

```
                          ┌─────────────────────────┐
                          │  No Plan Scheduled       │
                          │  (normal operation)      │
                          └──────────┬──────────────┘
                                     │ Governance passes
                                     │ SoftwareUpgradeProposal
                                     ▼
                          ┌─────────────────────────┐
                          │  Plan Scheduled          │
                          │  height=H, name=N        │
                          │  Blocks proceed normally  │
                          └──────────┬──────────────┘
                                     │ BlockHeight reaches H
                                     │ BeginBlocker fires
                                     ▼
                    ┌────────────────────────────────────┐
                    │  HasHandler(N)?                     │
                    └───────┬─────────────┬──────────────┘
                      NO    │             │ YES
                            ▼             ▼
               ┌──────────────────┐ ┌──────────────────┐
               │ Write            │ │ ApplyUpgrade()    │
               │ upgrade-info.json│ │ - Run handler     │
               │ PANIC:           │ │ - Update versions │
               │ UPGRADE NEEDED   │ │ - Increment proto │
               │                  │ │ - ClearPlan       │
               │ (crash loop)     │ │ - SetDone         │
               └───────┬──────────┘ └──────────┬───────┘
                       │                        │
                       │ Operator swaps binary  │
                       │ (or controller swaps   │
                       │  container image)       │
                       │                        │
                       ▼                        ▼
               ┌──────────────────┐ ┌──────────────────┐
               │ Restart with     │ │  Normal operation │
               │ new binary       │ │  (upgrade done)   │
               │ HasHandler = YES │ └──────────────────┘
               │ → ApplyUpgrade   │
               └───────┬──────────┘
                       │
                       ▼
               ┌──────────────────┐
               │  Normal operation │
               │  (upgrade done)   │
               └──────────────────┘
```
