---
title: "MemIAVL Internals"
component: sei-db/sc/memiavl
source: github.com/sei-protocol/sei-db@v0.0.53/sc/memiavl
origin: "Forked from Cronos MemIAVL (crypto-org-chain/cronos), Oct 2023"
purpose: "Memory-mapped IAVL tree for Sei's State Commitment (SC) layer"
updated: 2026-04-07
---

# MemIAVL Internals

MemIAVL is Sei's replacement for the standard cosmos/iavl database. It serves as the **State Commitment (SC) layer** in SeiDB's two-layer architecture (SC + SS). The core idea: keep the entire Merkle tree in memory via mmap, write only changesets to a WAL in real time, and periodically materialize full snapshots to disk.

Source: `github.com/sei-protocol/sei-db@v0.0.53/sc/memiavl/`

## 1. Architecture Overview

Standard IAVL persists every node update to a key-value database (LevelDB/RocksDB) on every block commit. MemIAVL inverts this:

1. **Real-time writes** go only to a Write-Ahead Log (WAL/changelog) -- compact changeset records, not full tree nodes.
2. **Periodic snapshots** (default every 10,000 blocks) materialize the full tree to flat files on disk.
3. **Snapshot files are mmap-ed** into memory, giving the OS page cache control over what stays resident.
4. **Between snapshots**, in-memory `MemNode` objects overlay the mmap-ed `PersistedNode` base.

### Directory Layout

```
<db-dir>/
  current -> snapshot-00000000000000010000   # symlink to active snapshot
  snapshot-00000000000000010000/
    __metadata                               # protobuf: CommitInfo + InitialVersion
    bank/                                    # one sub-dir per cosmos module/store
      metadata    # 12 bytes: magic(4) + format(4) + version(4)
      nodes       # fixed-size branch node array (48 bytes each)
      leaves      # fixed-size leaf node array (48 bytes each)
      kvs         # variable-length key-value pairs, sequentially packed
    acc/
      metadata
      nodes
      leaves
      kvs
    ...
  rlog/                                      # WAL directory (tidwall/wal segments)
  LOCK                                       # exclusive file lock for writers
```

## 2. Node Structure

MemIAVL has two node types that implement a shared `Node` interface:

### Node Interface

```go
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

### MemNode (In-Memory)

`MemNode` is a heap-allocated Go struct used for uncommitted/recently-modified nodes:

```go
type MemNode struct {
    height  uint8
    size    int64
    version uint32
    key     []byte
    value   []byte   // nil for branch nodes
    left    Node     // can point to MemNode or PersistedNode
    right   Node
    hash    []byte   // lazily computed, cleared on mutation
}
```

- **Leaf**: `height == 0`, has `key` + `value`, no children.
- **Branch**: `height > 0`, has `key` (smallest key in right subtree), `left`/`right` children, no value.
- **Hash**: computed lazily on first access, cleared (`nil`) when node is mutated.
- **Copy-on-Write (COW)**: `Mutate(version, cowVersion)` clones the node only if `node.version <= cowVersion`, otherwise modifies in place. This enables safe snapshots via `tree.Copy()`.

### PersistedNode (Mmap-backed)

`PersistedNode` reads directly from mmap-ed byte buffers with zero deserialization:

```go
type PersistedNode struct {
    snapshot *Snapshot
    isLeaf   bool
    index    uint32
}
```

It delegates all field access to the underlying `Snapshot`'s byte arrays. Two separate fixed-size file formats exist:

**Branch node layout (48 bytes in `nodes` file):**
```
height    : 1 byte
preTrees  : 1 byte   (pending subtree count, used for snapshot encoding/decoding)
_padding  : 2 bytes
version   : 4 bytes (uint32, little-endian)
size      : 4 bytes (uint32, little-endian)  -- number of leaf descendants
key_leaf  : 4 bytes (uint32, little-endian)  -- leaf index of smallest key in right subtree
hash      : 32 bytes (SHA-256)
```

**Leaf node layout (48 bytes in `leaves` file):**
```
version    : 4 bytes (uint32, little-endian)
key_len    : 4 bytes (uint32, little-endian)
key_offset : 8 bytes (uint64, little-endian)  -- byte offset into kvs file
hash       : 32 bytes (SHA-256)
```

**KVs file format:**
```
[key_len: 4 bytes LE][key bytes][value_len: 4 bytes LE][value bytes]
*repeated for each leaf, in sorted key order*
```

### Key Difference: No Pointers in Persisted Format

Standard IAVL stores each node with explicit left/right hash references. MemIAVL eliminates stored child pointers entirely. Instead, child indices are **derived from the post-order traversal layout**:

```
right child index = self index - 1
left child index  = key_leaf - preTrees - 2
start leaf        = index + 2 - size + preTrees
end leaf          = index + preTrees + 1
```

This is possible because nodes are written in depth-first post-order, so the tree structure is implicit in the array positions. This saves significant space and eliminates pointer-chasing on reads.

### Native Byte Order Optimization

Two build variants exist via build tags:
- **Default (`layout_little_endian.go`)**: Uses `binary.LittleEndian` to decode fields from raw byte slices.
- **`nativebyteorder` tag (`layout_native.go`)**: Uses `unsafe.Pointer` to cast mmap-ed memory directly to Go structs, achieving true zero-copy field access. Requires little-endian CPU (verified at init time).

## 3. Tree Structure

```go
type Tree struct {
    version        uint32
    root           Node        // nil for empty tree
    snapshot       *Snapshot   // the base mmap-ed snapshot, if any
    initialVersion uint32
    cowVersion     uint32      // copy-on-write boundary version
    zeroCopy       bool
    mtx            *sync.RWMutex
}
```

### Write Path

1. `Tree.Set(key, value)` or `Tree.Remove(key)` calls `setRecursive` / `removeRecursive`.
2. These walk the tree from root, calling `node.Mutate(version, cowVersion)` at each visited node.
3. `PersistedNode.Mutate()` creates a new `MemNode` copying the persisted data.
4. `MemNode.Mutate()` either clones (if `version <= cowVersion`) or modifies in place.
5. After insertion/deletion, the tree is rebalanced via `rotateLeft`/`rotateRight` (standard AVL rotations).
6. Hash is cleared on mutation; lazily recomputed when needed.

### Read Path

1. `Tree.Get(key)` calls `root.Get(key)`.
2. `PersistedNode.Get()` uses **binary search** over the leaf array in the snapshot (O(log n) with zero-copy key comparisons against mmap-ed data). This is much faster than tree traversal.
3. `MemNode.Get()` does standard recursive BST lookup.
4. If `zeroCopy == false`, returned slices are cloned to avoid retaining mmap-ed memory.

### Iteration

The `Iterator` uses an explicit stack-based DFS traversal (not recursion). It pushes/pops `Node` interface values, which can be a mix of `PersistedNode` and `MemNode`.

### Version Management

- Versions are `uint32` (max ~4.29 billion blocks).
- `SaveVersion(updateHash)` increments the version counter. Hash computation is optional.
- There is no multi-version tree in memory. Only the latest version exists. Historical queries are served by:
  - Loading a specific snapshot + replaying WAL entries up to the target version.
  - Or delegating to the SS (State Store) layer for raw key-value history.

## 4. Hashing Scheme

MemIAVL uses **the exact same hashing algorithm as standard cosmos/iavl**, ensuring hash compatibility:

```go
func writeHashBytes(node Node, w io.Writer) error {
    // varint: height, size, version
    // if leaf: length-prefixed key + SHA256(value)
    // if branch: length-prefixed left_hash + right_hash
}
```

- Hash algorithm: SHA-256.
- Leaf hash input: `varint(height=0) || varint(size=1) || varint(version) || len_prefix(key) || len_prefix(SHA256(value))`.
- Branch hash input: `varint(height) || varint(size) || varint(version) || len_prefix(left_hash) || len_prefix(right_hash)`.
- Length prefixes use `uvarint` encoding.
- Value is pre-hashed with SHA-256 before inclusion in the leaf hash (enables value-less proofs).

The test suite (`tree_test.go`) verifies hash compatibility by computing reference hashes with the standard `cosmos/iavl` library and comparing them against MemIAVL output.

## 5. Snapshots

### Snapshot Creation (`Tree.WriteSnapshot`)

1. Creates three output files: `nodes`, `leaves`, `kvs`, plus a `metadata` file.
2. Traverses the tree recursively in **depth-first post-order** (left, right, self).
3. Leaf nodes write: key-value to `kvs`, then leaf record to `leaves`.
4. Branch nodes write their record to `nodes` after both children are written.
5. The `preTrees` field records the count of pending (unresolved) subtrees at write time, enabling child index reconstruction during reads.
6. All files are `fsync`-ed before writing metadata.
7. Metadata is written last (12 bytes: magic "IAVL" + format 0 + version).
8. Uses `bufio.Writer` with 64MB buffer for I/O efficiency.

### Snapshot Loading (`OpenSnapshot`)

1. Reads and validates the `metadata` file (magic number, format version).
2. Mmap-s the `nodes`, `leaves`, and `kvs` files with `PROT_READ, MAP_SHARED, MADV_RANDOM`.
3. Validates file sizes: `len(nodes) % 48 == 0`, `len(leaves) % 48 == 0`, and `branches + 1 == leaves` (or both zero).
4. Root node is the last branch node (highest index), since post-order means root is written last.
5. Returns a `Snapshot` struct that wraps the mmap handles.

### MultiTree Snapshots

Each cosmos module has its own independent MemIAVL tree. `MultiTree` manages them together:
- All trees share the same version.
- Snapshot writes are parallelized per tree using a `pond.WorkerPool`.
- A protobuf `__metadata` file stores `CommitInfo` (per-store root hashes and versions) plus `InitialVersion`.

### Background Snapshot Rewrite

At configurable intervals (default 10,000 blocks):
1. The current `MultiTree` is copied (COW).
2. A background goroutine writes the snapshot to `snapshot-N-tmp/`.
3. On completion, renamed to `snapshot-N/` and `current` symlink updated atomically.
4. The new snapshot is loaded via mmap.
5. The changelog is replayed from the snapshot version to catch up.
6. The main tree switches to the new mmap-backed base.
7. Old snapshots are pruned (keeping `SnapshotKeepRecent`, default 1).
8. The WAL is truncated before the earliest remaining snapshot.

## 6. Write-Ahead Log (Changelog)

The WAL uses `tidwall/wal` (a segmented, append-only log) to persist changesets:

```go
type ChangelogEntry struct {
    Version    int64
    Changesets []*NamedChangeSet   // per-module key-value changes
    Upgrades   []*TreeNameUpgrade  // module add/delete/rename
}
```

- Each block commit appends one entry to the WAL.
- Entries are protobuf-encoded.
- Async writing is supported via a buffered channel (default buffer: 100 entries).
- The WAL is indexed by `offset = version - initialVersion + 1` (1-indexed).
- On startup, the WAL is replayed from the snapshot version to rebuild the in-memory tree.
- WAL truncation happens after snapshot pruning: entries before the earliest snapshot are removed.

### Crash Recovery

If the node crashes mid-commit:
1. On restart, the latest snapshot is loaded (mmap).
2. The WAL is replayed from the snapshot version forward.
3. If the crash happened during WAL write, `tidwall/wal` handles corrupt tail detection and truncation.
4. The tree converges to the last fully committed version.

## 7. Import/Export (State Sync)

### Export

`MultiTreeExporter` iterates through all trees in order:
1. Emits the tree name (string).
2. Emits all nodes in post-order via `Exporter`.
3. If the tree is backed by a snapshot at the export version, a fast sequential scan is used (iterating the files linearly). Otherwise, a recursive post-order traversal is used.

Each exported node is a `SnapshotNode`:
```go
type SnapshotNode struct {
    Key     []byte
    Value   []byte   // nil for branch nodes
    Version int64
    Height  int8     // 0 for leaves
}
```

### Import

`MultiTreeImporter` reconstructs each tree:
1. Receives tree names and nodes in post-order.
2. For each tree, a `TreeImporter` runs a goroutine that consumes nodes from a channel.
3. Leaf nodes are written directly to the snapshot files.
4. Branch nodes use a stack-based algorithm: pop the last two entries from the leaves/nodes stacks, compute the hash, write the branch, push the result.
5. After all nodes are imported, creates the `__metadata` file with `CommitInfo`.
6. Renames `snapshot-N-tmp/` to `snapshot-N/` and updates the `current` symlink.
7. Sets `InitialVersion = height + 1` so subsequent WAL entries start at index 1.

## 8. Copy-on-Write (COW) Semantics

`Tree.Copy()` creates a logical snapshot for concurrent reads:

1. If the root is a `MemNode`, sets `cowVersion = currentVersion`.
2. Returns a new `Tree` sharing the same root pointer.
3. Subsequent mutations on the original tree clone any `MemNode` with `version <= cowVersion` before modifying.
4. This enables safe concurrent reads on the copy while the original continues to be mutated.
5. `PersistedNode.Mutate()` always creates a new `MemNode` (they are inherently immutable).

## 9. Proof Generation

MemIAVL generates **ICS-23 compatible proofs** for both membership and non-membership:

- `GetMembershipProof(key)`: walks root to leaf collecting `PathToLeaf` (inner node hashes), then converts to ICS-23 `ExistenceProof`.
- `GetNonMembershipProof(key)`: finds the adjacent keys via index lookup, generates existence proofs for both neighbors, wraps in `NonExistenceProof`.
- The proof format is identical to standard IAVL, using the same `IavlSpec`.
- Historical proofs are only available for versions that haven't been pruned (i.e., versions between the earliest snapshot and the current version).

## 10. Performance Characteristics

### Why MemIAVL is Faster

| Dimension | Standard IAVL | MemIAVL |
|-----------|--------------|---------|
| **Write amplification** | Every commit writes all modified nodes to LevelDB (each node ~100+ bytes, plus LevelDB compaction) | Only changesets written to WAL in real time (much smaller). Full snapshot writes happen infrequently. |
| **Read amplification** | Multiple LevelDB reads to traverse from root to leaf (each level = separate DB lookup) | Single mmap page fault loads the needed data. Binary search over leaf array for persisted trees. Read amplification = 1. |
| **Space amplification** | Every version retains full node set; orphan tracking required | Only latest tree on disk. Changesets are 1:100 ratio vs full nodes. Historical data delegated to SS layer. |
| **Commit latency** | Synchronous LevelDB writes of all dirty nodes | WAL append only. Hash computation optional (can be deferred). Async commit supported. |
| **Memory** | Node cache (LRU) with limited hit rates | OS page cache manages mmap-ed files automatically. Hot data stays resident. |

### Benchmark Context (from benchmark_test.go)

The benchmark compares random Get on 1M items across:
- `memiavl` (in-memory tree)
- `memiavl-disk` (mmap-backed, no app cache)
- `btree` (tidwall B-tree, degree 2 and 32)
- `iavl-lru` (standard IAVL LRU cache)
- `go-map` (native hashmap)
- `binary-search` (sorted slice)

The mmap-backed PersistedNode.Get uses binary search over the sorted leaf array, which is competitive with a Go map for cache-hot data and degrades gracefully when data exceeds RAM (page faults instead of OOM).

### SeiDB Headline Numbers (from README)

- 60% reduction in active chain state size
- ~90% reduction in historical data growth rate
- 1200% improvement in state sync times
- 287x improvement in block commit times
- 2x overall TPS improvement

## 11. Tradeoffs and Limitations

1. **Memory pressure**: When state size significantly exceeds available RAM, performance degrades to disk I/O speed (mmap page faults). Standard IAVL with a tuned LRU cache can be more predictable in memory-constrained environments.

2. **Historical proofs are slow**: Only versions between the earliest snapshot and current are provable. Proofs for very old versions require replaying changesets from genesis (or a sufficiently old snapshot).

3. **Snapshot creation is heavy**: Writing a full snapshot is an expensive I/O operation. During snapshot write, the COW copy holds all in-memory mutations. The background goroutine and worker pool mitigate this, but it's still a periodic spike.

4. **Version overflow**: Versions are `uint32`, limiting the chain to ~4.29 billion blocks. The README notes this "should be enough in foreseeable future."

5. **No concurrent writers**: The DB uses a single mutex for all writes. Reads are concurrent via `RWMutex` on individual trees, but commits are serialized.

6. **Zero-copy lifetime**: When `ZeroCopy` is enabled, returned byte slices point directly into mmap-ed memory. If the snapshot is closed/replaced (e.g., after background rewrite), accessing those slices causes a segfault. Callers must not retain zero-copy values beyond the current block execution.

7. **Crash during snapshot rewrite**: If the process crashes during a background snapshot rewrite, the incomplete `snapshot-N-tmp/` directory is cleaned up on next startup. The WAL ensures no data loss.

## 12. Key Configuration

From `config.StateCommitConfig`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `SnapshotInterval` | 10,000 | Blocks between snapshot rewrites |
| `SnapshotKeepRecent` | 1 | Old snapshots to retain (excluding latest) |
| `AsyncCommitBuffer` | 100 | WAL async write queue depth. <=0 = sync. |
| `ZeroCopy` | false | Return mmap pointers directly (faster but unsafe to retain) |
| `SnapshotWriterLimit` | 1 | Concurrent goroutines for snapshot writing |
| `CacheSize` | 100,000 | Deprecated; relies on OS page cache via mmap |

## 13. Relationship to sei-iavl

`sei-iavl` (`github.com/sei-protocol/sei-iavl@v0.2.0`) is a **fork of cosmos/iavl v0.19.4**. It is the standard IAVL implementation with minor Sei-specific patches. It is used:

- As a reference for hash compatibility testing (the test suite computes hashes with both implementations and asserts equality).
- For the `iavl.ChangeSet` and `iavl.KVPair` types used throughout MemIAVL's changeset handling.
- For the `iavl.PathToLeaf` / `iavl.ProofInnerNode` types used in proof generation.

MemIAVL is **not** a modification of sei-iavl. It is a completely separate implementation that produces identical Merkle hashes, enabling a seamless swap of the storage backend without breaking consensus.
