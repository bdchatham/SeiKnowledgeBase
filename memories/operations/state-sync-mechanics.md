---
topic: "State sync end-to-end code path in sei-tendermint"
sources:
  - repo: sei-tendermint
    version: v0.6.4
    files:
      - internal/statesync/reactor.go
      - internal/statesync/syncer.go
      - internal/statesync/snapshots.go
      - internal/statesync/block_queue.go
      - internal/statesync/chunks.go
      - light/stateprovider.go
      - light/verifier.go
      - light/client.go
      - node/node.go
      - config/config.go
      - internal/blocksync/reactor.go
verified: 2026-04-07
confidence: high
---

## Overview

State sync restores a node to a recent height from a snapshot without replaying the full chain history. The pipeline is strictly linear: **snapshot restore -> header backfill -> block sync -> consensus**. No blocks are executed during the state sync or backfill phases — block execution only begins during block sync at `snapshot_height + 1`.

## Entry Decision (node/node.go)

State sync activates when ALL of these hold:
- `cfg.StateSync.Enable == true`
- Node is not the sole validator
- `state.LastBlockHeight == 0` (fresh node with no existing state)

```go
stateSync := cfg.StateSync.Enable && !onlyValidatorIsUs(state, pubKey)
if stateSync && state.LastBlockHeight > 0 {
    logger.Info("Found local state with non-zero height, skipping state sync")
    stateSync = false
}
```

State sync is a one-shot operation. Once a node has any state (`LastBlockHeight > 0`), it will never re-enter state sync — even on restart. The handshake is also skipped during state sync (`node.shouldHandshake = !stateSync`).

## Phase 1: Snapshot Discovery and Application (reactor.go, syncer.go)

`Reactor.Sync()` orchestrates the full process:

1. **Wait for peers** — blocks until at least 2 peers are connected (light client needs cross-verification)
2. **Initialize state provider** — P2P-based (peers serve light blocks) or RPC-based (trusted RPC servers), configured by `UseP2P`
3. **Discover snapshots** — broadcasts `SnapshotsRequest` to all peers, waits `DiscoveryTime` (default 15s)
4. **Rank snapshots** — `snapshotPool.Ranked()` prefers: highest height > most peers > highest format. Snapshots with above-median peer support are preferred.
5. **Offer to ABCI app** — `OfferSnapshot` RPC. App can accept, reject, reject format, or reject sender.
6. **Fetch chunks** — `N` concurrent fetcher goroutines (default 4) pull chunks from peers
7. **Apply chunks** — sequential `ApplySnapshotChunk` ABCI calls. App controls flow: can request refetches, reject senders, retry individual chunks, abort.
8. **Verify app** — calls `Info()` on ABCI app, checks 3 things:
   - `resp.AppVersion == state.Version.Consensus.App`
   - `resp.LastBlockAppHash == snapshot.trustedAppHash`
   - `resp.LastBlockHeight == snapshot.Height`
9. **Bootstrap stores** — `stateStore.Bootstrap(state)` and `blockStore.SaveSeenCommit()`

### Where the trusted app hash comes from (stateprovider.go)

```go
func (s *stateProviderRPC) AppHash(ctx context.Context, height uint64) ([]byte, error) {
    // Fetch header at height+1 — it contains the app hash for height
    header, err := s.verifyLightBlockAtHeight(ctx, height+1, time.Now())
    // Also pre-fetch height+2 to avoid race condition
    _, err = s.verifyLightBlockAtHeight(ctx, height+2, time.Now())
    return header.AppHash, nil
}
```

The app hash is extracted from the block header at `snapshot_height + 1`, which in Tendermint convention contains the state root after committing block at `snapshot_height`. This header is verified by the light client back to the trust anchor.

### What the light client actually verifies (verifier.go)

The light client does **purely cryptographic verification** — no block execution:

- `VerifyAdjacent`: checks header expiry, basic validation, `ValidatorsHash == previous NextValidatorsHash`, 2/3+ validator signatures
- `VerifyNonAdjacent` (skipping mode): checks 1/3+ of trusted validators signed the new header, plus 2/3+ of new validators signed it
- A `LightBlock` is just `{SignedHeader, ValidatorSet}` — no transactions

Whether the trust height is 100 blocks or 100,000 blocks behind the snapshot, the light client only verifies more signatures. It never touches the ABCI app.

## Phase 2: Backfill (reactor.go)

After snapshot application, historical light blocks (headers + validator sets, NOT full blocks) are fetched backward:

```go
func (r *Reactor) Backfill(ctx context.Context, state sm.State) error {
    stopHeight := state.LastBlockHeight - r.cfg.BackfillBlocks
    stopTime := state.LastBlockTime.Add(-r.cfg.BackfillDuration)
    // ...
}
```

- Controlled by `BackfillBlocks` (count) and `BackfillDuration` (time)
- Uses `blockQueue` with concurrent fetchers walking backward
- Single verifier thread validates hash chain: each block's hash must match the next block's `LastBlockID`
- Stores signed headers and validator sets only — no block execution

## Phase 3: Block Sync (blocksync/reactor.go)

The post-sync hook transitions to block sync:

```go
postSyncHook := func(ctx context.Context, state sm.State) error {
    csReactor.SetStateSyncingMetrics(0)
    csReactor.SetBlockSyncingMetrics(1)
    return bcReactor.SwitchToBlockSync(ctx, state)
}
```

Block sync fetches **full blocks** starting from `snapshot_height + 1` and executes them through the ABCI app (`DeliverTx`/`FinalizeBlock`). This is the first point where any block execution occurs. Once caught up to the network tip, the node calls `SwitchToConsensus()`.

## State Bootstrapped After Sync (state/store.go)

`Bootstrap()` saves minimal state at the snapshot height:

| Data | Heights Saved | What's Missing |
|------|--------------|----------------|
| Validators | H-1, H, H+1 | All prior validator sets |
| Consensus params | H + LastHeightChanged | Historical param changes |
| App version | Single value at H | No version transition history |
| Blocks | None (until backfill/block-sync) | Everything before H |

## Network Upgrades and State Sync

State sync works across upgrade boundaries. Key findings:

- **Snapshots are binary-agnostic at the Tendermint level** — the `Snapshot` protobuf has `{Height, Format, Chunks, Hash, Metadata}` with no version field. Version compatibility is the ABCI app's responsibility via `OfferSnapshot`.
- **Running the latest binary works** — if the snapshot is past the upgrade height, the state contains post-migration data. The binary never needs to replay through upgrade heights.
- **The trust height does NOT cause block re-execution** — even if set before the upgrade, the light client only verifies signatures, never executes blocks. The trust height/period is purely a security parameter.
- **App hash mismatches** during state sync mean the snapshot data and block headers disagree. Common causes: snapshot peer running a different binary, snapshot taken at exact upgrade boundary, non-deterministic IAVL restore.

## Key Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `statesync.enable` | false | Master switch |
| `statesync.rpc_servers` | [] | RPC endpoints for light client (required if UseP2P=false) |
| `statesync.trust_height` | (required) | Height of trusted block anchor |
| `statesync.trust_hash` | (required) | Hash of trusted block header |
| `statesync.trust_period` | 168h | Max age of trusted block — should be < unbonding period |
| `statesync.discovery_time` | 15s | Time to discover snapshots before attempting restore |
| `statesync.chunk_request_timeout` | 15s | Timeout per chunk request |
| `statesync.fetchers` | 4 | Concurrent chunk/block fetchers |
| `statesync.use_p2p` | false | Use P2P layer vs RPC for light block verification |
| `statesync.use_local_snapshot` | false | Use local snapshots only (no peer discovery) |
| `statesync.backfill_blocks` | 0 | Historical blocks to backfill after sync |
| `statesync.backfill_duration` | 0s | Time-based backfill cutoff |

## Key Takeaways

- State sync is one-shot: `LastBlockHeight > 0` prevents re-entry. No incremental or delta snapshots.
- No blocks are executed during state sync — the snapshot is a full state restore, the light client is pure signature verification.
- The trust height/period is a security parameter only. It does not affect which blocks are processed or which app hash is expected.
- App hash mismatches always mean the snapshot peer's state diverges from what validators committed in block headers. Debug by comparing the snapshot peer's `/abci_info` against validator block headers.
- The trust period should be less than the chain's unbonding period (typically 21 days). Setting it to 9999h weakens light client security but does not cause functional issues.
- After state sync, the transition is always: snapshot restore -> backfill headers -> block sync (first execution) -> consensus. There is no shortcut.
