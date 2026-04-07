---
title: Sei Chain Upgrade Orchestration
category: consensus
confidence: high
sources:
  sei-cosmos:
    version: v0.3.66
    files:
      - x/upgrade/abci.go                    # BeginBlocker — the core halt/apply logic
      - x/upgrade/keeper/keeper.go           # Keeper: plan storage, upgrade-info.json, handler registry
      - x/upgrade/types/plan.go              # Plan type, ShouldExecute, UpgradeDetails, IsMinorRelease
      - x/upgrade/types/proposal.go          # SoftwareUpgradeProposal, CancelSoftwareUpgradeProposal
      - x/upgrade/types/upgrade.pb.go        # Proto-generated Plan struct (Name, Height, Info)
      - x/upgrade/types/handler.go           # UpgradeHandler func signature
      - x/upgrade/types/storeloader.go       # UpgradeStoreLoader for store migrations
      - x/upgrade/handler.go                 # Gov proposal handler (ScheduleUpgrade / ClearUpgradePlan)
      - x/upgrade/module.go                  # AppModule — BeginBlock wiring, GRPC service registration
      - x/upgrade/doc.go                     # Comprehensive doc on upgrade workflow
      - x/upgrade/client/cli/tx.go           # CLI: submit-upgrade-proposal
      - x/upgrade/keeper/grpc_query.go       # GRPC: CurrentPlan, AppliedPlan, ModuleVersions
      - baseapp/abci.go                      # Commit() — halt-height mechanism (SIGINT/SIGTERM)
      - baseapp/baseapp.go                   # haltHeight field, halt() method
      - server/start.go                      # StartCmd restart loop, halt-height flag, skip-upgrades flag
      - server/util.go                       # WaitForQuitSignals, RestartErrorCode=100, restartCh
      - server/config/config.go              # BaseConfig.HaltHeight, HaltTime
      - server/in_place.go                   # InPlaceTestnetCreator
  sei-chain:
    version: v0.0.38
    files:
      - app/upgrades.go                      # upgradesList, RegisterUpgradeHandlers, RunMigrations
      - app/upgrades_test.go                 # Tests for override list
      - app/upgrade_test.go                  # Tests for upgrade plan, skip optimistic processing
      - app/app.go                           # Wiring: UpgradeKeeper, HardForkManager, SetStoreUpgradeHandlers
      - app/abci.go                          # BeginBlock delegates to BaseApp (which calls module manager)
      - app/upgrades/fork_manager.go         # HardForkManager — height+chainID-targeted handlers in BeginBlock
      - app/upgrades/v0/upgrade.go           # Example hard fork handler (wasm migration)
  sei-tendermint:
    version: v0.6.4
    files:
      - config/config.go                     # SelfRemediationConfig (restart cooldown, blocks-behind)
      - node/public.go                       # NewDefault, New — restartCh plumbing
      - node/node.go                         # makeNode — restartCh passed to reactors
      - internal/blocksync/reactor.go        # autoRestartIfBehind — self-remediation restart
tags: [upgrade, governance, binary-swap, halt-height, cosmovisor, hard-fork, migration]
---

# Sei Chain Upgrade Orchestration

## 1. Two Upgrade Mechanisms

Sei has **two distinct upgrade mechanisms** that run in the same binary:

### 1a. Governance-Driven Upgrades (x/upgrade module)
Standard Cosmos SDK upgrade path. A `SoftwareUpgradeProposal` is submitted via governance, specifying a `Plan` with a **name** and **target height**. When the chain reaches that height, the node halts (panics) and must be restarted with a new binary that has a matching upgrade handler.

### 1b. Hard Fork Manager (sei-chain specific)
A Sei-specific `HardForkManager` that executes arbitrary handlers at specific heights for specific chain IDs, without requiring a governance proposal. Runs in `BeginBlock` *before* the module manager's BeginBlock. Used for emergency state migrations (e.g., wasm contract migrations). Does NOT halt the node — it runs inline.

---

## 2. Governance Upgrade Flow (x/upgrade)

### 2.1 Proposal Submission
```
seid tx gov submit-proposal software-upgrade <name> \
  --upgrade-height <height> \
  --upgrade-info '<json>' \
  --title "..." --description "..." --deposit <coins>
```

The proposal wraps a `types.Plan`:
```go
type Plan struct {
    Name   string    // Unique upgrade name (e.g., "v5.3.0")
    Height int64     // Block height at which upgrade executes
    Info   string    // JSON metadata (e.g., {"upgradeType":"minor"} or cosmovisor download URL)
}
```

### 2.2 Proposal Passes
When the governance vote passes, the gov module calls `upgrade.NewSoftwareUpgradeProposalHandler`, which calls `keeper.ScheduleUpgrade(ctx, plan)`. The plan is persisted to the upgrade module's KV store.

### 2.3 Cancellation
A `CancelSoftwareUpgradeProposal` can be submitted and voted on. If passed, calls `keeper.ClearUpgradePlan()`.

---

## 3. What Happens at the Upgrade Height

### 3.1 The BeginBlocker Decision Tree

At every block, `upgrade.BeginBlocker` runs (it is registered as the **second** BeginBlocker after epoch, but before all other modules):

```
BeginBlock(height N):
  1. Get plan from store
  2. If no plan → return (normal block)
  3. If plan.Height <= N (ShouldExecute):
     a. If skip-upgrade flag set for this height → clear plan, continue
     b. If handler registered for plan.Name → applyUpgrade() → continue normally
     c. If NO handler registered → panicUpgradeNeeded() → NODE HALTS
  4. If plan.Height > N (not yet time):
     a. Parse plan.Info for UpgradeDetails
     b. If minor release → log scheduled message, return (no halt, no panic)
     c. If NOT minor AND handler IS registered → PANIC ("BINARY UPDATED BEFORE TRIGGER!")
```

### 3.2 The Halt Mechanism: PANIC (not graceful exit)

**Critical for controller design**: When an upgrade is needed and no handler exists, the node **panics**:

```go
func panicUpgradeNeeded(k keeper.Keeper, ctx sdk.Context, plan types.Plan) {
    // Write upgrade-info.json to disk BEFORE panicking
    err := k.DumpUpgradeInfoWithInfoToDisk(ctx.BlockHeight(), plan.Name, plan.Info)
    // ...
    panic("UPGRADE \"<name>\" NEEDED at height <N>: <info>")
}
```

The panic:
- Is caught by the Cosmos SDK/Tendermint runtime
- Prevents the block from being committed
- The node process does NOT cleanly exit — it crashes
- The process supervisor (systemd, K8s, cosmovisor) restarts it
- On restart with the SAME binary, it will panic again at the same height → **restart loop**
- On restart with the NEW binary (which has the handler), `applyUpgrade()` runs instead

### 3.3 What applyUpgrade() Does

```go
func (k Keeper) ApplyUpgrade(ctx sdk.Context, plan types.Plan) {
    handler := k.upgradeHandlers[plan.Name]
    updatedVM, err := handler(ctx, plan, k.GetModuleVersionMap(ctx))
    // ^ Runs migrations via app.mm.RunMigrations()
    k.SetModuleVersionMap(ctx, updatedVM)
    k.setProtocolVersion(ctx, nextProtocolVersion)
    k.versionSetter.SetProtocolVersion(nextProtocolVersion)
    k.ClearIBCState(ctx, plan.Height)
    k.ClearUpgradePlan(ctx)
    k.SetDone(ctx, plan.Name)
}
```

Sequence: run handler → update module versions → increment protocol version → clear plan → mark done.

---

## 4. upgrade-info.json

### 4.1 Location
Written to `$HOME/data/upgrade-info.json` (e.g., `/root/.sei/data/upgrade-info.json`).

### 4.2 Contents
```json
{
  "name": "v5.3.0",
  "height": 12345678,
  "info": "{\"upgradeType\":\"minor\"}"
}
```

### 4.3 When Written
Written by `panicUpgradeNeeded()` **right before the panic**. This is critical — it happens before the node crashes.

### 4.4 Who Reads It
1. **cosmovisor**: Reads `upgrade-info.json` to determine which binary to switch to
2. **The app itself on restart**: `ReadUpgradeInfoFromDisk()` is called by `SetStoreUpgradeHandlers()` during app initialization to configure store loaders for any store migrations needed by the upgrade

---

## 5. How Cosmovisor Works

Cosmovisor is an external process wrapper (not part of sei-chain itself). Its integration points:

1. **Watches for panics**: Monitors the seid process. When it exits/panics, reads `upgrade-info.json`
2. **Binary layout**: Expects binaries at `$DAEMON_HOME/cosmovisor/upgrades/<name>/bin/<daemon>`
3. **Plan.Info auto-download**: If `Plan.Info` contains a download URL in the cosmovisor format, it can auto-download the binary
4. **Swap**: Symlinks `current` to the upgrade directory
5. **Restart**: Starts the new binary

**For the controller**: Cosmovisor is not needed if the controller manages the binary swap. The controller just needs to:
- Detect the panic / upgrade-info.json
- Replace the container image
- Restart the pod

---

## 6. halt-height vs Upgrade Height

### 6.1 halt-height (App Configuration)
A **separate mechanism** from governance upgrades. Set via:
- `--halt-height` CLI flag
- `halt-height` in `app.toml`

Checked in `BaseApp.Commit()` AFTER the block is committed:
```go
case app.haltHeight > 0 && uint64(header.Height) >= app.haltHeight:
    halt = true
```

When triggered, calls `app.halt()` which sends **SIGINT then SIGTERM** to its own process:
```go
func (app *BaseApp) halt() {
    p, _ := os.FindProcess(os.Getpid())
    sigIntErr := p.Signal(syscall.SIGINT)
    sigTermErr := p.Signal(syscall.SIGTERM)
    // Fallback: os.Exit(0)
}
```

**Key difference from upgrade panic**:
- halt-height: Block IS committed, THEN the node gracefully exits via signals
- Upgrade panic: Block is NOT committed, node crashes via panic
- halt-height: No upgrade-info.json written
- halt-height: No restart loop — the node just stops

### 6.2 When halt-height Is Used
Operators can use halt-height to manually coordinate upgrades without governance. They:
1. Set halt-height to the agreed block
2. Node halts after committing that block
3. Swap binary manually
4. Restart (the node does NOT re-halt because the height has passed)

---

## 7. Missed Upgrade Height — Can a Node Catch Up?

### 7.1 If the node was offline during the upgrade
The node can catch up IF it is started with the **new binary**. Here's why:

- The upgrade plan is stored on-chain
- When the node replays blocks during catch-up, it will hit the upgrade height
- If the handler is registered (new binary), `applyUpgrade()` runs during replay
- If the handler is NOT registered (old binary), it panics during replay

### 7.2 Downgrade Verification
On startup, `BeginBlocker` checks `GetLastCompletedUpgrade()`. If the last completed upgrade doesn't have a handler in the current binary, it panics:
```
"Wrong app version %d, upgrade handler is missing for %s upgrade plan"
```
This prevents running an old binary against state that has already been upgraded.

### 7.3 The --unsafe-skip-upgrades Escape Hatch
```
seid start --unsafe-skip-upgrades 12345678,12345679
```
This populates `skipUpgradeHeights` in the keeper. When the upgrade height is reached, instead of halting or applying, it **clears the plan** and continues with the current binary. Used for emergency social consensus to bypass a broken upgrade.

---

## 8. Upgrade Handlers and State Migrations

### 8.1 Registration in sei-chain
All upgrades are registered in `app/upgrades.go`:

```go
var upgradesList = []string{
    "1.0.2beta", "1.0.3beta", ..., "v5.3.0",
}

func (app App) RegisterUpgradeHandlers() {
    for _, upgradeName := range upgradesList {
        app.UpgradeKeeper.SetUpgradeHandler(upgradeName, func(ctx sdk.Context, plan upgradetypes.Plan, fromVM module.VersionMap) (module.VersionMap, error) {
            return app.mm.RunMigrations(ctx, app.configurator, fromVM)
        })
    }
}
```

Key observations:
- **All versions use the same handler**: `RunMigrations` with the module manager
- The list must be **alphabetically sorted** (enforced at startup with `log.Fatal`)
- Can be overridden via `UPGRADE_VERSION_LIST` env var (for testing)
- Only `1.2.3beta` has a special case (setting CommunityTax to 0)

### 8.2 Store Upgrades
When an upgrade adds/removes/renames KV stores, `SetStoreUpgradeHandlers()` configures the store loader:

```go
func (app *App) SetStoreUpgradeHandlers() {
    upgradeInfo, _ := app.UpgradeKeeper.ReadUpgradeInfoFromDisk()
    if upgradeInfo.Name == "v5.1.0" && !app.UpgradeKeeper.IsSkipHeight(upgradeInfo.Height) {
        storeUpgrades := storetypes.StoreUpgrades{
            Added: []string{evmtypes.StoreKey},
        }
        app.SetStoreLoader(upgradetypes.UpgradeStoreLoader(upgradeInfo.Height, &storeUpgrades))
    }
}
```

This runs during app initialization (before the first block), reading from `upgrade-info.json`.

### 8.3 RunMigrations
The module manager's `RunMigrations` compares `fromVM` (stored versions) with each module's current `ConsensusVersion()`. For any module where the version has changed, it runs registered migration functions. This is how schema changes are applied.

---

## 9. The Exact Upgrade Sequence

### Phase 1: Proposal and Scheduling
```
1. Operator submits SoftwareUpgradeProposal (name="v5.4.0", height=N)
2. Validators vote during governance period
3. Proposal passes → keeper.ScheduleUpgrade(plan) stores plan in state
4. Plan is now on-chain, visible via `seid q upgrade plan`
```

### Phase 2: Approaching Upgrade Height (blocks before N)
```
5. Every BeginBlock: plan found, but ShouldExecute returns false (height < N)
6. If minor release: logs "UPGRADE SCHEDULED" every 100 blocks
7. If major release AND new binary already running: PANIC ("BINARY UPDATED BEFORE TRIGGER")
   - This prevents running the new binary too early
8. Optimistic processing is skipped when plan.ShouldExecute is true
```

### Phase 3: Upgrade Height Reached (block N)
```
9.  BeginBlock at height N: plan.ShouldExecute(ctx) returns true
10. OLD binary: no handler registered → panicUpgradeNeeded()
    a. Writes upgrade-info.json to $HOME/data/
    b. Logs: UPGRADE "v5.4.0" NEEDED at height N
    c. panic() — node crashes, block N is NOT committed
11. Process supervisor restarts node → same binary → same panic → restart loop
```

### Phase 4: Binary Swap
```
12. Operator (or cosmovisor, or controller) replaces binary/image with v5.4.0
13. New binary starts up
14. App initialization:
    a. RegisterUpgradeHandlers() — registers handler for "v5.4.0"
    b. SetStoreUpgradeHandlers() — reads upgrade-info.json, configures store loader
    c. Store loader runs if this upgrade needs store changes (add/remove/rename stores)
```

### Phase 5: Migration and Resumption
```
15. BeginBlock at height N (replay/new block):
    a. plan.ShouldExecute(ctx) returns true
    b. Handler IS registered → applyUpgrade()
    c. handler() calls RunMigrations → module-level state migrations execute
    d. Module version map is updated in store
    e. Protocol version is incremented
    f. Plan is cleared, upgrade marked as "done"
16. Rest of BeginBlock runs normally
17. Block N commits successfully
18. Once 2/3+ of validators upgrade, consensus resumes
19. Chain is live on the new version
```

---

## 10. Sei-Specific Modifications

### 10.1 Minor Release Support (Sei-cosmos addition)
Standard Cosmos SDK does not have minor/major release distinction. Sei-cosmos added:

```go
type UpgradeDetails struct {
    UpgradeType string `json:"upgradeType"`
}

func (ud UpgradeDetails) IsMinorRelease() bool {
    return strings.EqualFold(ud.UpgradeType, "minor")
}
```

If `Plan.Info` contains `{"upgradeType":"minor"}`, the upgrade:
- Does NOT panic if the handler is missing before the target height
- CAN be applied early if the handler is present
- Still executes at the scheduled height

### 10.2 Hard Fork Manager (Sei-chain addition)
Separate from x/upgrade. Runs in `app.BeginBlocker()` before the module manager:

```go
func (app *App) BeginBlocker(ctx sdk.Context, req abci.RequestBeginBlock) abci.ResponseBeginBlock {
    if app.HardForkManager.TargetHeightReached(ctx) {
        app.HardForkManager.ExecuteForTargetHeight(ctx)
    }
    return app.mm.BeginBlock(ctx, req)
}
```

Properties:
- Handlers are registered per chain-ID and target height
- No governance needed — baked into the binary
- Panics on error (not recoverable)
- Used for emergency migrations (e.g., wasm contract migration on specific chain)

### 10.3 Self-Remediation Restart Loop (Sei-tendermint addition)
Sei-tendermint adds a `restartCh` mechanism. The `server.StartCmd` runs a **restart loop**:

```go
exitCode := RestartErrorCode  // 100
for {
    err = startInProcess(...)
    errCode, ok := err.(ErrorCode)
    exitCode = errCode.Code
    if exitCode != RestartErrorCode {
        break
    }
    serverCtx.Logger.Info("restarting node...")
    canRestartAfter = time.Now().Add(restartCoolDownDuration)
}
```

Reactors (blocksync, pex, statesync) can send to `restartCh` to trigger a graceful in-process restart. This is **separate from upgrade panics** — it's for self-remediation when the node falls behind or loses peers.

Self-remediation config (`config.toml [self-remediation]`):
- `blocks-behind-threshold`: Restart if node falls this many blocks behind peers (0=disabled)
- `blocks-behind-check-interval-seconds`: How often to check (default 60s)
- `restart-cooldown-seconds`: Minimum time between restarts (default 600s / 10min)
- `p2p-no-peers-available-window-seconds`: Restart if no peers for this long (0=disabled)
- `statesync-no-peers-available-window-seconds`: Restart if no statesync peers (0=disabled)

### 10.4 UPGRADE_VERSION_LIST Override
For integration testing, the upgrade list can be overridden via environment variable:
```
UPGRADE_VERSION_LIST="v5.2.0,v5.3.0" seid start
```

### 10.5 Optimistic Processing Skip
Sei skips optimistic block processing when an upgrade is imminent:
```go
plan, found := app.UpgradeKeeper.GetUpgradePlan(ctx)
if found && plan.ShouldExecute(ctx) {
    app.optimisticProcessingInfo.Aborted = true
}
```

---

## 11. Controller Design Implications

### 11.1 Detecting an Upgrade
The controller can detect a pending upgrade by:
1. **GRPC query**: `cosmos.upgrade.v1beta1.Query/CurrentPlan` returns the scheduled plan (name, height)
2. **REST**: `/cosmos/upgrade/v1beta1/current_plan`
3. **Node logs**: `UPGRADE "<name>" NEEDED at height <N>` (after the panic)
4. **upgrade-info.json**: File appears in `$HOME/data/` after the first panic
5. **Pod crash loop**: The pod will enter CrashLoopBackOff due to panic restart loop

### 11.2 Performing the Upgrade
The controller should:
1. Detect the scheduled upgrade height BEFORE it arrives (query the plan)
2. Prepare the new image (pre-pull, validate)
3. At upgrade height: pods will panic and CrashLoopBackOff
4. Update the StatefulSet image to the new version
5. Pods restart with new binary → migrations run → chain resumes

### 11.3 Key Timing Constraints
- **Do NOT update the image before the upgrade height for major upgrades**: The new binary will panic with "BINARY UPDATED BEFORE TRIGGER" if it has the handler registered and the height hasn't been reached yet
- **Minor upgrades** (upgradeType=minor) can be applied early — the handler just runs when available
- **After the upgrade height**: Update the image ASAP, as every restart with the old binary is a wasted cycle (panic → restart → panic)
- **Rollout order doesn't matter**: Each node independently applies the upgrade at the same height. There's no leader election or coordination needed between nodes

### 11.4 halt-height as an Alternative
For non-governance upgrades (e.g., hotfixes), the controller can:
1. Set `halt-height` in app.toml to a coordinated height
2. Node commits the block at that height, then exits cleanly (SIGINT)
3. Update the image
4. Remove the halt-height config
5. Restart — no panic, no upgrade-info.json, just a clean restart with new binary

### 11.5 Emergency: --unsafe-skip-upgrades
If an upgrade is broken, the controller can restart nodes with:
```
seid start --unsafe-skip-upgrades <height>
```
This skips the upgrade at that height and continues with the current binary.

### 11.6 Process Exit Codes
- **Panic (upgrade needed)**: Process exits non-zero (crash). K8s sees container crash.
- **halt-height**: Process exits 0 (clean SIGINT/SIGTERM). K8s sees container complete.
- **Self-remediation restart**: Error code 100 (RestartErrorCode). Handled in-process, not visible to K8s.
- **SIGINT/SIGTERM**: Exit code 128 + signal number.

### 11.7 Querying Upgrade State
```bash
# Check for pending upgrade
seid q upgrade plan

# Check if an upgrade was already applied
seid q upgrade applied <name>

# List module versions (useful for migration verification)
seid q upgrade module_versions
```

GRPC equivalents:
- `cosmos.upgrade.v1beta1.Query/CurrentPlan`
- `cosmos.upgrade.v1beta1.Query/AppliedPlan`
- `cosmos.upgrade.v1beta1.Query/ModuleVersions`
