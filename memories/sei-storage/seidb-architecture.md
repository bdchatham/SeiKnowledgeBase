---
title: "SeiDB Architecture: Two-Layer Storage Model"
category: storage
confidence: verified-from-source
sources:
  sei-db:
    version: "v0.0.53"
    module: "github.com/sei-protocol/sei-db"
    files_read:
      - config/config.go
      - README.md
      - sc/store.go
      - sc/types/tree.go
      - sc/types/store.go
      - sc/types/snapshot.go
      - sc/memiavl/db.go
      - sc/memiavl/tree.go
      - sc/memiavl/node.go
      - sc/memiavl/mem_node.go
      - sc/memiavl/persisted_node.go
      - sc/memiavl/snapshot.go
      - sc/memiavl/multitree.go
      - sc/memiavl/mmap.go
      - sc/memiavl/opts.go
      - sc/memiavl/import.go
      - sc/memiavl/export.go
      - sc/memiavl/layout_native.go
      - sc/memiavl/layout_little_endian.go
      - sc/memiavl/README.md
      - ss/store.go
      - ss/types/store.go
      - ss/pebbledb/db.go
      - ss/pebbledb/comparator.go
      - ss/pebbledb/batch.go
      - ss/pebbledb/iterator.go
      - ss/pruning/manager.go
      - common/utils/path.go
  sei-cosmos:
    version: "v0.3.66"
    module: "github.com/sei-protocol/sei-cosmos"
    files_read:
      - storev2/rootmulti/store.go
      - storev2/state/store.go
      - storev2/commitment/store.go
  sei-chain:
    version: "v0.0.38"
    module: "github.com/sei-protocol/sei-chain"
    files_read:
      - app/seidb.go
  sei-iavl:
    version: "v0.2.0"
    module: "github.com/sei-protocol/sei-iavl"
    files_read:
      - FORKED_CHANGELOG.md
      - mutable_tree.go
last_updated: "2026-04-07"
---

# SeiDB Architecture

SeiDB is Sei's custom storage engine that replaces the standard Cosmos IAVL+LevelDB backend. It splits storage into two purpose-built layers: **State Commitment (SC)** using MemIAVL for Merkle proofs and active state, and **State Store (SS)** using PebbleDB for versioned historical queries. The design is inspired by the Cosmos StoreV2 ADR-065.

## Performance Claims (from README)

- 60% reduction in active chain state size
- ~90% reduction in historical data growth rate
- 1200% improvement in state sync times, 2x block sync improvement
- 287x improvement in block commit times
- 2x overall TPS improvement
- Archive nodes achieve full-node performance levels

---

## 1. SC/SS Two-Layer Architecture

### Why Two Layers?

Standard Cosmos uses a single IAVL tree (backed by LevelDB/GoLevelDB) for everything: active state, Merkle hashing, historical state, and proofs. This causes:

- **Write amplification**: Every commit rewrites many IAVL nodes to the DB engine
- **State bloat**: Historical versions accumulate rapidly on disk
- **Slow commits**: Merkle hash computation + DB persistence on the critical path

SeiDB decouples these concerns:

| Concern | SC Layer (MemIAVL) | SS Layer (PebbleDB) |
|---|---|---|
| **Purpose** | Merkle tree, active state, app hash | Versioned historical key-value queries |
| **Data model** | Memory-mapped IAVL tree | MVCC-encoded key-value pairs |
| **Pruning** | Aggressive (snapshots only) | Configurable retention (default 100K blocks) |
| **Proofs** | Yes (ICS23 commitment proofs) | No |
| **Backend** | Custom flat files + mmap | PebbleDB (or RocksDB) |

### Data Flow on Commit

From `storev2/rootmulti/store.go`, the commit path:

1. `commitment.Store` collects changesets (Set/Delete calls append to `changeSet.Pairs`)
2. `flush()` pops changesets from all stores, sorts by name, feeds them to:
   - SC: `scStore.ApplyChangeSets(changeSets)` then `scStore.Commit()`
   - SS: Pushed to `pendingChanges` channel, consumed asynchronously by `StateStoreCommit()` goroutine
3. SC commit returns new version; SS writes happen in background

```go
// storev2/rootmulti/store.go - Commit path
func (rs *Store) Commit(bumpVersion bool) types.CommitID {
    rs.flush()                    // Apply changesets to SC + queue to SS
    rs.scStore.Commit()           // Bump version, write changelog, maybe snapshot
    // SS writes happen async in StateStoreCommit() goroutine
}
```

### Historical Queries

From `storev2/rootmulti/store.go`:

- **Latest version**: Reads directly from SC (MemIAVL tree in memory)
- **Historical version without proof**: Reads from SS (PebbleDB versioned lookup)
- **Historical version with proof**: Opens a read-only SC store at that version, loads snapshot + replays changelog

```go
func (rs *Store) CacheMultiStoreWithVersion(version int64) {
    // Non-proof queries -> SS store
    stores[k] = state.NewStore(rs.ssStore, k, version)
}

func (rs *Store) Query(req abci.RequestQuery) {
    if !req.Prove && rs.ssStore != nil {
        store = state.NewStore(rs.ssStore, ..., version)  // SS for data
    } else {
        scStore, _ := rs.scStore.LoadVersion(version, true) // SC for proofs
        store = commitment.NewStore(scStore.GetTreeByName(storeName), ...)
    }
}
```

---

## 2. State Commitment (SC) Layer: MemIAVL

### What is MemIAVL?

MemIAVL (forked from Cronos's memiavl implementation) keeps the entire IAVL Merkle tree in memory, backed by memory-mapped flat files. Instead of persisting individual tree nodes to a database engine on every commit, it:

1. Writes only **changesets** (deltas) to a Write-Ahead Log (changelog) on each block
2. Takes periodic **snapshots** of the full tree to disk (default every 10,000 blocks)
3. On startup, loads the latest snapshot via mmap, then replays the changelog to catch up

### Node Types

MemIAVL has two node types implementing a common `Node` interface:

```go
// sc/memiavl/node.go
type Node interface {
    Height() uint8
    IsLeaf() bool
    Size() int64
    Version() uint32
    Key() []byte
    Value() []byte
    Left() Node
    Right() Node
    Hash() []byte
    SafeHash() []byte
    Mutate(version, cowVersion uint32) *MemNode
    Get(key []byte) ([]byte, uint32)
    GetByIndex(uint32) ([]byte, []byte)
}
```

**MemNode** -- in-memory nodes for uncommitted/modified state:

```go
// sc/memiavl/mem_node.go
type MemNode struct {
    height  uint8
    size    int64
    version uint32
    key     []byte
    value   []byte
    left    Node
    right   Node
    hash    []byte
}
```

**PersistedNode** -- zero-copy references into mmap'd snapshot files:

```go
// sc/memiavl/persisted_node.go
type PersistedNode struct {
    snapshot *Snapshot
    isLeaf   bool
    index    uint32  // index into the nodes or leaves array
}
```

PersistedNode does NOT store data itself. It reads directly from the mmap'd byte slices via computed offsets. When a PersistedNode is modified, `Mutate()` converts it to a MemNode by copying the key/value data from the snapshot:

```go
func (node PersistedNode) Mutate(version, _ uint32) *MemNode {
    if node.isLeaf {
        key, value := node.snapshot.LeafKeyValue(node.index)
        return &MemNode{height: 0, size: 1, version: version, key: key, value: value}
    }
    // ... similar for branch nodes
}
```

### Copy-on-Write (COW)

The tree uses a `cowVersion` to implement copy-on-write semantics. When a tree is copied (for background snapshot), the `cowVersion` is set. Subsequent mutations only clone nodes whose version <= cowVersion:

```go
func (node *MemNode) Mutate(version, cowVersion uint32) *MemNode {
    n := node
    if node.version <= cowVersion {
        cloned := *node
        n = &cloned
    }
    n.version = version
    n.hash = nil
    return n
}
```

### Binary Search on PersistedNode

A key performance optimization: `PersistedNode.Get()` performs a **binary search** over the leaf array rather than traversing the tree:

```go
func (node PersistedNode) Get(key []byte) ([]byte, uint32) {
    // binary search in the leaf node array
    i := uint32(sort.Search(int(count), func(i int) bool {
        leafKey := node.snapshot.LeafKey(start + uint32(i))
        return bytes.Compare(leafKey, key) >= 0
    }))
    // ...
}
```

This is O(log n) on the flat sorted leaf array, avoiding pointer chasing through the tree structure.

### MultiTree

MemIAVL manages multiple trees (one per Cosmos module) through `MultiTree`:

```go
// sc/memiavl/multitree.go
type MultiTree struct {
    initialVersion uint32
    zeroCopy       bool
    cacheSize      int
    trees          []NamedTree       // ordered by name
    treesByName    map[string]int    // name -> index
    lastCommitInfo proto.CommitInfo
    metadata       proto.MultiTreeMetadata
}

type NamedTree struct {
    *Tree
    Name string
}
```

All trees share the same version number. Snapshots are created atomically for all trees at the same height.

### Zero-Copy Mode

When `ZeroCopy` is enabled, `Get()` and `Iterator()` return byte slices that point directly into mmap'd buffers. This eliminates memory allocation for reads, but the slices must NOT be retained beyond the current block execution. The SDK address cache is disabled in this mode.

---

## 3. MemIAVL vs Standard cosmos/iavl

| Aspect | Standard IAVL (cosmos/iavl) | MemIAVL (sei-db) |
|---|---|---|
| **Storage** | All nodes persisted to LevelDB/GoLevelDB as KV pairs | Tree held in memory; snapshots as flat mmap files |
| **Commit** | Writes every modified/orphaned node to DB | Writes only changeset to WAL (changelog) |
| **Write amplification** | High (many small KV writes per block) | Minimal (compact changeset + periodic snapshot) |
| **Read amplification** | Multiple DB lookups to traverse tree | 1 (direct mmap offset access) |
| **Versions** | All versions in same DB, pruned by deleting old nodes | Snapshot at version N, changelog for deltas |
| **Node format** | Protobuf-encoded, variable size | Fixed-size binary layout (48 bytes branch, 48 bytes leaf) |
| **Orphan tracking** | Explicit orphan set in DB | Not needed (snapshots replace entire tree) |
| **Memory** | DB cache + OS page cache | Full tree in memory/mmap + OS page cache |
| **Startup** | Load root from DB | Load snapshot (mmap) + replay changelog |

Standard IAVL (sei-iavl v0.2.0, forked from cosmos/iavl v0.19.4) uses a `MutableTree` backed by `nodeDB` which reads/writes nodes to a `tm-db` backend. Every `SaveVersion` persists changed nodes and records orphans. MemIAVL eliminates this entirely by treating the tree as an in-memory data structure with periodic disk materialization.

---

## 4. PebbleDB in the SS Layer

### What Goes Where

- **MemIAVL (SC)**: Active Merkle tree, app hash computation, ICS23 proofs, current state reads during block execution
- **PebbleDB (SS)**: All versioned key-value pairs for historical queries, archive node support

### MVCC Key Encoding

PebbleDB uses a custom MVCC (Multi-Version Concurrency Control) comparator to store versioned keys:

```
Key format:   <store_prefix><key>\x00[<version_big_endian_8bytes>]<version_length_byte>
Value format: <value>\x00[<tombstone_version_big_endian_8bytes>]<tombstone_length_byte>
```

The store prefix is `s/k:<storeKey>/`, e.g., `s/k:bank/`.

```go
// ss/pebbledb/comparator.go
func MVCCEncode(key []byte, version int64) (dst []byte) {
    dst = append(dst, key...)
    dst = append(dst, 0)          // separator
    if version != 0 {
        extra := byte(1 + 8)      // version length indicator
        dst = encodeUint64Ascending(dst, uint64(version))
        dst = append(dst, extra)
    }
    return dst
}
```

Deletions are recorded as **tombstones** -- the value is set to `"TOMBSTONE"` with the deletion version encoded in the value's MVCC suffix. A `Get()` at version V returns nil if the tombstone version <= V.

### PebbleDB Configuration

```go
// ss/pebbledb/db.go
opts := &pebble.Options{
    Cache:                       pebble.NewCache(32 MB),
    Comparer:                    MVCCComparer,
    L0CompactionThreshold:       2,
    L0StopWritesThreshold:       1000,
    LBaseMaxBytes:               64 MB,
    Levels:                      7 levels,
    MaxConcurrentCompactions:    3,
    MemTableSize:                64 MB,
    MemTableStopWritesThreshold: 4,
}
// All levels use Zstd compression, 32KB block size, 256KB index block, bloom filters
// Bottom level (L6) has no bloom filter
```

### Versioned Get

```go
func getMVCCSlice(db *pebble.DB, storeKey string, key []byte, targetVersion int64) ([]byte, error) {
    // SeekLT to find the largest version <= targetVersion for this key
    // Then check tombstone to determine if key was deleted
}
```

### Async Writes

SS supports async writes via a dedicated changelog:
1. Changesets are written to WAL first
2. A background goroutine (`writeAsyncInBackground`) reads from `pendingChanges` channel and applies to PebbleDB
3. On startup, `RecoverStateStore` replays the WAL from the last committed SS version

---

## 5. Snapshot Mechanics

### SC Snapshots (MemIAVL)

Snapshots are periodic full materializations of the in-memory tree to disk. The `DB.Commit()` method triggers them:

```go
func (db *DB) rewriteIfApplicable(height int64) {
    if height % int64(db.snapshotInterval) != 0 { return }
    db.rewriteSnapshotBackground()
}
```

**Background snapshot rewrite process:**

1. `Copy()` the current MultiTree (COW -- sets cowVersion so modifications don't affect the copy)
2. In a goroutine: write the copy to a new `snapshot-{version}-tmp` directory
3. Rename to `snapshot-{version}`, update `current` symlink
4. On next `Commit()`, `checkBackgroundSnapshotRewrite()` picks up the result
5. Load the new snapshot via mmap, catch up remaining changelog entries
6. Atomically switch the active MultiTree to the new snapshot-backed tree
7. Prune old snapshots (keeping `snapshotKeepRecent` old ones)

### Snapshot File Format

Each module tree produces 4 files:

**metadata** (12 bytes):
```
magic:   uint32 = 0x4C564149 ("IAVL" in little-endian)
format:  uint32 = 0
version: uint32
```

**nodes** (branch nodes, 48 bytes each):
```
height:    1 byte
preTrees:  1 byte
_padding:  2 bytes
version:   uint32
size:      uint32
key_leaf:  uint32    (index of smallest leaf in right subtree)
hash:      [32]byte
```

**leaves** (leaf nodes, 48 bytes each):
```
version:    uint32
key_len:    uint32
key_offset: uint64   (offset into kvs file)
hash:       [32]byte
```

**kvs** (key-value pairs, variable length):
```
key_len:   uint32
key:       [key_len]byte
value_len: uint32
value:     [value_len]byte
(repeat for each leaf)
```

Nodes are written in **depth-first post-order** traversal. The root is always the last branch node. Child indices are inferred from position:
- `right child index = self index - 1`
- `left child index = key_leaf - preTrees - 2`

### SS Snapshots (State Sync)

During Tendermint state sync, both SC and SS are populated in parallel:

```go
// storev2/rootmulti/store.go - restore()
scImporter, _ := rs.scStore.Importer(height)
ssImporter = make(chan sstypes.SnapshotNode, 10000)
go rs.ssStore.Import(height, ssImporter)

for { // read snapshot stream
    // Feed IAVL nodes to SC importer
    scImporter.AddNode(node)
    // Feed leaf key-values to SS importer
    if node.Height == 0 {
        ssImporter <- sstypes.SnapshotNode{StoreKey: storeKey, Key: node.Key, Value: node.Value}
    }
}
```

---

## 6. Pruning

### SC Pruning

SC pruning is snapshot-based. After a new snapshot is written:

1. Old snapshots beyond `snapshotKeepRecent` are deleted via `atomicRemoveDir()` (rename to `-tmp` then `rm -rf`)
2. Changelog entries before the earliest remaining snapshot are truncated

```go
func (db *DB) pruneSnapshots() {
    counter := db.snapshotKeepRecent
    traverseSnapshots(db.dir, false, func(version int64) {
        if version >= currentVersion { return }  // skip newer/current
        if counter > 0 { counter--; return }     // keep recent ones
        atomicRemoveDir(filepath.Join(db.dir, snapshotName(version)))
    })
    // Truncate changelog before earliest snapshot
    db.streamHandler.TruncateBefore(earliestVersion + 1)
}
```

Default: keep 1 old snapshot + current = 2 total snapshots on disk.

### SS Pruning

SS uses a background `PruningManager`:

```go
// ss/pruning/manager.go
func (m *Manager) Start() {
    go func() {
        for {
            pruneVersion := latestVersion - m.keepRecent
            m.stateStore.Prune(pruneVersion)
            // Random jitter on sleep interval to avoid thundering herd
            time.Sleep(pruneInterval + randomDelay)
        }
    }()
}
```

The PebbleDB `Prune()` method iterates all keys:
- For each key with version <= pruneVersion, if a newer version of the same key exists (also <= pruneVersion), the older version is deleted
- Tombstoned keys with version <= pruneVersion are deleted
- If `KeepLastVersion` is false, all versions <= pruneVersion are deleted (even the last one for a key)
- Uses `storeKeyDirty` map to skip modules that haven't been updated since the last prune

Default: keep 100,000 blocks, prune every 600 seconds.

---

## 7. Performance Characteristics

### Why SeiDB is Faster

1. **Write amplification reduction**: Standard IAVL writes multiple nodes (branch + orphans) per key change to LevelDB. MemIAVL writes only the compact changeset to a sequential WAL. Snapshot rewrites happen infrequently (every 10K blocks) and in the background.

2. **Async commit**: The changelog write is the only synchronous operation. SC tree updates happen in memory. SS writes are fully async via a channel.

3. **Read performance**: PersistedNode reads are single mmap dereference (offset math, no DB lookup). MemNode reads are pointer traversal in Go heap. Binary search over sorted leaf arrays for PersistedNode.Get().

4. **Zero-copy reads**: With `ZeroCopy=true`, no memory allocation for reads from mmap'd data.

5. **Sequential I/O**: WAL and snapshot files are written sequentially, optimal for SSDs. No random seeks for writes.

6. **Reduced space**: SS stores raw key-values (no Merkle tree overhead). SC snapshots are compact flat files. Historical growth is managed by pruning.

7. **Parallel snapshot writes**: `MultiTree.WriteSnapshot()` uses a worker pool to write each module's snapshot files concurrently.

### Trade-offs

- **Memory pressure**: The full active state must fit in memory (or rely on mmap paging). Performance degrades if state exceeds physical RAM.
- **Historical proofs are slower**: Require loading a full SC snapshot + replaying changelog to the target version.
- **Snapshot creation is heavy**: Writing multi-GB flat files periodically. Mitigated by background execution + COW semantics.
- **No historical proof integrity validation**: SS store lacks Merkle proof capabilities.

---

## 8. Configuration

### app.toml Flags (from sei-chain app/seidb.go)

**State Commit (SC):**

| Flag | Config Key | Default | Description |
|---|---|---|---|
| `state-commit.sc-enable` | `Enable` | `true` | Enable SeiDB (false = fallback to IAVL) |
| `state-commit.sc-directory` | `Directory` | `""` (home dir) | SC data directory |
| `state-commit.sc-zero-copy` | `ZeroCopy` | `false` | Return mmap slices directly (no copy) |
| `state-commit.sc-async-commit-buffer` | `AsyncCommitBuffer` | `100` | Async commit queue size (<=0 = sync) |
| `state-commit.sc-keep-recent` | `SnapshotKeepRecent` | `1` | Old snapshots to retain |
| `state-commit.sc-snapshot-interval` | `SnapshotInterval` | `10000` | Blocks between snapshots |
| `state-commit.sc-snapshot-writer-limit` | `SnapshotWriterLimit` | `1` | Concurrent snapshot writers |
| `state-commit.sc-cache-size` | `CacheSize` | `100000` | *Deprecated* (relies on mmap page cache) |

**State Store (SS):**

| Flag | Config Key | Default | Description |
|---|---|---|---|
| `state-store.ss-enable` | `Enable` | `true` | Enable SS for historical queries |
| `state-store.ss-db-directory` | `DBDirectory` | `""` (home dir) | SS data directory |
| `state-store.ss-backend` | `Backend` | `"pebbledb"` | Backend: pebbledb, rocksdb |
| `state-store.ss-async-write-buffer` | `AsyncWriteBuffer` | `100` | Async write queue (<=0 = sync) |
| `state-store.ss-keep-recent` | `KeepRecent` | `100000` | Versions to retain (0 = keep all) |
| `state-store.ss-prune-interval` | `PruneIntervalSeconds` | `600` | Seconds between prune runs |
| `state-store.ss-import-num-workers` | `ImportNumWorkers` | `1` | Goroutines for state sync import |

**Validation rule**: When snapshot export interval > 0 and SC is enabled, SS must also be enabled (otherwise panic on startup).

---

## 9. Data Directory Layout

Given `$HOME` as the node home directory:

```
$HOME/data/
  committer.db/                    # SC (MemIAVL) directory
    LOCK                           # File lock (prevents concurrent access)
    current -> snapshot-0000000000000050000   # Symlink to latest snapshot
    snapshot-0000000000000050000/   # Snapshot at block 50000
      __metadata                   # Protobuf: CommitInfo + InitialVersion
      bank/                        # Module tree snapshot
        metadata                   # 12 bytes: magic + format + version
        nodes                      # Fixed-size branch nodes (48 bytes each, mmap'd)
        leaves                     # Fixed-size leaf nodes (48 bytes each, mmap'd)
        kvs                        # Packed key-value pairs (variable length, mmap'd)
      acc/                         # Another module
        metadata
        nodes
        leaves
        kvs
      staking/
        ...
      wasm/
        ...
    snapshot-0000000000000040000/   # Previous snapshot (if snapshotKeepRecent >= 1)
      ...
    changelog/                     # Write-ahead log for changesets
      ...                          # Sequential entries: version + changesets + upgrades
  pebbledb/                        # SS (PebbleDB) directory
    ...                            # Standard PebbleDB files (SSTs, WAL, MANIFEST, etc.)
    changelog/                     # Optional dedicated SS changelog (if DedicatedChangelog=true)
```

Snapshot directory names are zero-padded to 20 digits: `snapshot-XXXXXXXXXXXXXXXXXXXX`.

---

## 10. Historical Query Support

### Two-Layer Query Routing

The `storev2/rootmulti/store.go` routes queries based on whether proofs are needed:

**Without proofs (most queries):**
- Route to SS (PebbleDB)
- SS wraps the `StateStore` in a `state.Store` adapter at the requested version
- PebbleDB performs MVCC lookup: `SeekLT(key, version+1)` to find the latest value for that key at or before the requested version
- Tombstone-aware: if the value has a tombstone <= requested version, returns nil

**With proofs (IBC, light client):**
- Route to SC (MemIAVL)
- Opens a read-only `CommitStore` at the requested version
- Loads snapshot + replays changelog to reconstruct the tree at that height
- Generates ICS23 commitment proofs from the reconstructed tree
- This is expensive (potentially replaying thousands of changelog entries)

### SS Read Path Detail

```go
// ss/pebbledb/db.go
func (db *Database) Get(storeKey string, targetVersion int64, key []byte) ([]byte, error) {
    // Construct MVCC key and SeekLT to find latest version <= target
    prefixedVal := getMVCCSlice(db.storage, storeKey, key, targetVersion)
    // Split value into actual value + tombstone
    valBz, tombBz, _ := SplitMVCCKey(prefixedVal)
    // If tombstone exists and tombstone version <= targetVersion, key is deleted
    if tombstone <= targetVersion { return nil }
    return valBz
}
```

### SS Iterator

The PebbleDB iterator (`ss/pebbledb/iterator.go`) implements version-aware iteration:
- Uses `SeekLT(key, version+1)` to position at the correct version for each key
- Skips tombstoned entries automatically
- Supports both forward and reverse iteration
- Handles version boundaries correctly when keys have been written at versions higher than requested

### XOR Hash Ranges

SS computes XOR hashes over configurable block ranges (default 1M blocks) for data integrity verification between nodes. This runs in the background after async writes, using 10 worker goroutines to parallelize hash computation across modules.

---

## Integration Summary

```
                    Block Execution
                         |
                    ChangeSet (Set/Delete per module)
                         |
                    +----+----+
                    |         |
                    v         v (async, via channel)
               SC Layer    SS Layer
            (MemIAVL)    (PebbleDB)
                |              |
         +------+------+      +-------+
         |      |      |      |       |
       Tree   WAL   Snapshot  MVCC  Prune
      (memory) (seq)  (mmap)  (KV)  (background)
         |
      App Hash
```

The key insight: SC handles the consensus-critical path (app hash, proofs) with memory-speed operations, while SS handles the query-heavy path (historical data, archive) with a purpose-built versioned KV store. Neither layer is burdened with the other's concerns.
