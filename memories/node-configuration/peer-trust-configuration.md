---
title: "Sei Node P2P Peer Discovery, Trust, and Connection Configuration"
created: 2026-04-07
sources:
  sei-tendermint:
    version: v0.6.4
    files:
      - config/config.go (P2PConfig struct, defaults)
      - config/toml.go (config.toml template, TOML key names)
      - internal/p2p/peermanager.go (PeerManagerOptions, scoring, retry, lifecycle)
      - internal/p2p/conn/secret_connection.go (STS handshake, ed25519 auth)
      - internal/p2p/transport_mconn.go (MConn transport, handshake flow)
      - internal/p2p/router.go (accept/dial/handshake peer, filtering)
      - internal/p2p/pex/reactor.go (PEX reactor, peer exchange protocol)
      - internal/p2p/pex/doc.go (PEX overview)
      - types/node_info.go (NodeInfo, CompatibleWith validation)
      - types/node_id.go (NodeID = hex(sha256(ed25519PubKey)[:20]))
  sei-config:
    version: v0.0.8
    files:
      - config.go (P2PConfig fields, SeiConfig struct)
      - defaults.go (mode-specific P2P overrides)
      - types.go (NodeMode enum)
  seictl:
    version: v0.0.23
    files:
      - sidecar/tasks/peers.go (PeerDiscoverer, EC2TagsSource, StaticSource)
      - sidecar/tasks/statesync.go (StateSyncConfigurer, trust point resolution)
      - sidecar/tasks/genesis_peers.go (GenesisPeersSetter, S3 peers.json)
      - sidecar/tasks/tendermint.go (status/block response types)
      - sidecar/client/tasks.go (PeerSourceType constants)
  sei-node-controller:
    branch: main (networking)
    files:
      - api/v1alpha1/common_types.go (PeerSource, LabelPeerSource, EC2TagsPeerSource)
      - internal/controller/node/peers.go (reconcilePeers, label resolution)
      - internal/planner/planner.go (discoverPeersParams, plan building)
      - internal/planner/bootstrap.go (bootstrap plan with peer discovery)
      - internal/task/config.go (DiscoverPeersParams, PeerSourceParam)
      - internal/task/bootstrap_resources.go (bootstrap Job/Service generation)
---

# Sei Node P2P Peer Discovery, Trust, and Connection Configuration

## 1. Peer Categories in Tendermint

Sei uses a fork of Tendermint (sei-tendermint v0.6.4) which defines several distinct peer categories, each with different connection semantics and scoring.

### 1.1 persistent-peers

**Config key:** `p2p.persistent-peers` (config.toml uses hyphens)
**Struct field:** `P2PConfig.PersistentPeers` (mapstructure: `persistent-peers`)
**Format:** Comma-separated list of `nodeID@host:port`

Persistent peers are the primary mechanism for ensuring a node maintains connections to specific known peers. Key behaviors:

- **Score 254** (`PeerScorePersistent`): Second highest score, only below unconditional (255)
- **Retry with persistence:** Uses `MaxRetryTimePersistent` for retry cap (separate from regular peers)
- **Cannot be banned:** `peerInfo.Ban()` is a no-op for persistent/blocksync peers -- the code prints a warning and returns
- **Cannot be evicted for upgrades:** Higher-scored peers will not displace persistent peers
- **Dial failure tolerance:** Exponential backoff with jitter: `MinRetryTime * 2^failures + jitter`, capped at `MaxRetryTimePersistent`
- **Always reconnect:** The peer manager will always attempt to maintain connections to persistent peers

This is the field written by the sidecar's `discover-peers` and `set-genesis-peers` tasks.

### 1.2 bootstrap-peers

**Config key:** `p2p.bootstrap-peers`
**Struct field:** `P2PConfig.BootstrapPeers`
**Format:** Comma-separated `nodeID@host:port`

Bootstrap peers are added to the peer store on startup to seed initial peer discovery. Unlike persistent peers, they have no special scoring and are treated as regular peers after initial contact. Either bootstrap-peers or persistent-peers must be provided for peer discovery to work.

### 1.3 unconditional-peer-ids

**Config key:** `p2p.unconditional-peer-ids`
**Struct field:** `P2PConfig.UnconditionalPeerIDs`
**Format:** Comma-separated node IDs (no host:port)

Unconditional peers bypass connection limits entirely:

- **Score 255** (`PeerScoreUnconditional`): Highest possible score
- **Bypass MaxConnected:** `Accepted()` allows unconditional peers even when `MaxConnected + MaxConnectedUpgrade` is reached
- **Not counted in NumConnected():** The `NumConnected()` method explicitly skips unconditional peers
- **Always reconnected:** Like persistent, but with even higher priority
- **Use case:** Sentry nodes protecting validators; the validator connects unconditionally to its sentries

### 1.4 private-peer-ids

**Config key:** `p2p.private-peer-ids`
**Struct field:** `P2PConfig.PrivatePeerIDs`
**Format:** Comma-separated node IDs

Private peers are never gossiped to other peers via the PEX protocol:

- **PEX filtering:** The `Advertise()` method skips peers whose NodeID is in `PrivatePeers`
- **Still connectable:** Other peers can still connect to a private peer if they know its address
- **Use case:** Validator nodes that should not have their addresses broadcast to the network
- **Auto-marking:** If DNS resolution fails for a peer address, the router automatically marks that peer as private

### 1.5 blocksync-peers

**Config key:** `p2p.blocksync-peers`
**Struct field:** `P2PConfig.BlockSyncPeers`
**Format:** Comma-separated `nodeID@host:port`

A Sei-specific addition for peers used exclusively during block synchronization:

- **Score 254** (`PeerScorePersistent`): Same score tier as persistent peers
- **Cannot be banned:** Same protection as persistent peers
- **Dedicated purpose:** Only used during the block sync protocol

### 1.6 seeds (removed in sei-tendermint)

The traditional Tendermint `seeds` field and `seed_mode` flag do not appear in sei-tendermint's `P2PConfig`. Seed functionality has been replaced by `bootstrap-peers` and the PEX reactor.

## 2. Cryptographic Handshake and Trust

### 2.1 Node Identity

Each node has an ed25519 keypair stored in `config/node_key.json`. The **NodeID** is the hex-encoded first 20 bytes of the SHA-256 hash of the ed25519 public key:

```
NodeID = hex(pubkey.Address()) = hex(sha256(ed25519PubKey)[:20])
```

This is a 40-character lowercase hex string (e.g., `a1b2c3d4e5f6...`).

### 2.2 Secret Connection (STS Protocol)

Every peer connection uses the **Station-to-Station (STS)** protocol for authenticated encryption. The handshake in `MakeSecretConnection()`:

1. **Ephemeral key generation:** Both sides generate ephemeral Curve25519 keypairs for forward secrecy
2. **Ephemeral key exchange:** Both sides simultaneously send their ephemeral public keys
3. **Transcript construction:** A Merlin transcript accumulates the lower and upper ephemeral public keys (sorted lexicographically)
4. **Diffie-Hellman:** `X25519(localEphPriv, remoteEphPub)` computes the shared secret
5. **Key derivation:** HKDF-SHA256 derives separate send/receive symmetric keys and a challenge from the transcript
6. **Authentication:** Both sides sign the challenge with their persistent ed25519 key and exchange signatures over the encrypted channel
7. **Verification:** Each side verifies the remote signature against the remote public key. Connection is rejected if the remote key is not ed25519 or if signature verification fails.

The result is a `SecretConnection` that:
- Encrypts all traffic with **ChaCha20-Poly1305** AEAD
- Authenticates the remote peer's ed25519 public key
- Provides perfect forward secrecy via ephemeral Curve25519 keys
- Frames messages in 1028-byte frames (4-byte length + 1024-byte payload), each individually sealed

### 2.3 Handshake Validation

After the STS handshake, the router performs additional validation (`handshakePeer()`):

1. **NodeID verification:** `NodeIDFromPubKey(peerKey) == peerInfo.NodeID` -- the authenticated public key must match the self-reported node ID
2. **NodeInfo validation:** Checks moniker, channels, listen address format
3. **Network match:** `peerInfo.Network == nodeInfo.Network` -- peers must be on the same chain ID. Mismatched peers are **deleted from the peer store**
4. **Protocol compatibility:** `CompatibleWith()` checks Block protocol version matches and at least one shared channel
5. **Expected ID match:** For outbound connections, the handshaked peer must match the expected NodeID from the address

### 2.4 What "Trust" Means in P2P

"Trust" in the Tendermint P2P layer is strictly **cryptographic identity authentication**:

- The STS handshake proves the remote peer controls the ed25519 private key corresponding to their NodeID
- There is **no PKI or certificate authority** -- any valid ed25519 key can connect
- Trust is established by **configuration**: you trust peers whose NodeIDs you put in `persistent-peers` or `unconditional-peer-ids`
- The PEX protocol gossips addresses freely; connecting to a gossiped peer still requires the STS handshake

This is distinct from **state sync trust**, which involves trusting a block hash at a specific height (see Section 7).

## 3. Peer Discovery Mechanisms

### 3.1 PEX Reactor (Peer Exchange)

The PEX reactor (`pex.Reactor`) is the primary runtime peer discovery mechanism:

- **Enabled by:** `p2p.pex = true` (default: true)
- **Channel:** Channel 0x00 (PEX channel)
- **Protocol:** Periodically sends `PexRequest` to connected peers, receives `PexResponse` containing up to 100 `PexAddress` entries
- **Adaptive polling:** Starts fast, slows down as the peer store fills. At 95% capacity, polls every 10 minutes. Uses a proportional control mechanism based on the fraction of "new" peers received
- **Self-advertisement:** `Advertise()` includes the node's own `SelfAddress` so peers learn how to dial back
- **Private peer filtering:** Never gossips peers in the `PrivatePeers` set
- **Rate limiting:** Tracks `lastReceivedRequests` to prevent peers from sending requests too often (min 100ms between requests from the same peer)
- **Self-remediation:** If all available peers disconnect and `P2pNoPeersRestartWindowSeconds > 0`, the PEX reactor triggers a full router restart

### 3.2 Static Configuration

Peers configured via `persistent-peers` or `bootstrap-peers` in config.toml are added to the peer store at startup. The peer manager will continuously attempt to dial these peers.

### 3.3 Manual RPC

The `/dial_peers` RPC endpoint (when `unsafe = true`) allows adding peers at runtime.

## 4. Peer Scoring and Eviction

### 4.1 Score Hierarchy

```
PeerScoreUnconditional    = 255  (unconditional peers -- never evicted)
PeerScorePersistent       = 254  (persistent + blocksync peers)
MaxPeerScoreNotPersistent = 253  (cap for non-persistent peers)
DefaultMutableScore       = 243  (starting score for new regular peers)
```

### 4.2 Score Calculation (`peerInfo.Score()`)

1. If `FixedScore > 0`, return it (test/override mechanism)
2. If `Unconditional`, return 255
3. Start with `MutableScore` (243 for new peers), or `PeerScorePersistent` (254) for persistent/blocksync
4. **Add** `ConsecSuccessfulBlocks / 5` (block sync performance bonus)
5. **Subtract** dial failure penalty with time decay: `failures * e^(-0.1 * hoursSinceLastFailure)`
6. **Subtract** disconnection penalty with time decay: `(numDisconnections * e^(-0.1 * hoursSinceLastConnect)) / 3`
7. Cap at `MaxPeerScoreNotPersistent` (253) for non-persistent peers
8. Floor at 0

### 4.3 Eviction

When connected peers exceed `MaxConnected`:
- The peer manager finds the lowest-scored connected peer as an upgrade candidate
- Higher-scored incoming or dialed peers can claim the slot
- The lowest-scored peer is marked for eviction
- Up to `MaxConnectedUpgrade` additional probe connections are allowed simultaneously

### 4.4 Banning

`peerInfo.Ban()` sets `FixedScore = 0`, `MutableScore = 0`, `ConsecSuccessfulBlocks = 0`. **Persistent and blocksync peers cannot be banned** -- the call is a no-op with a warning.

### 4.5 Pruning

When the peer store exceeds `MaxPeers`, the lowest-scored unconnected, non-dialing peers are deleted.

## 5. Connection Limits and Rate Limiting

### 5.1 Connection Limits

| Parameter | Config Key | Default |
|-----------|-----------|---------|
| Max connections (in+out) | `p2p.max-connections` | 64 (tendermint), 100 (sei-config) |
| Max incoming connection attempts/IP | `p2p.max-incoming-connection-attempts` | 100 |
| Allow duplicate IP | `p2p.allow-duplicate-ip` | false |

Note: sei-config defaults differ from raw sei-tendermint defaults. The `max-connections` default is 100 in sei-config.

### 5.2 Mode-Specific Overrides

| Mode | max-connections | allow-duplicate-ip |
|------|----------------|-------------------|
| Full (default) | 100 | false |
| Validator | 100 | false |
| Seed | 1000 | true |
| Archive | 100 (inherits full) | false |

### 5.3 Rate Limiting

The `connTracker` in the router rate-limits incoming connection attempts per IP address:
- Window: `IncomingConnectionWindow` (default 100ms)
- Max attempts per window: `MaxIncomingConnectionAttempts` (default 100)

### 5.4 Peer Filtering

The router supports two filtering hooks:
- `FilterPeerByIP`: Called before handshake, filters by IP address
- `FilterPeerByID`: Called after handshake, filters by NodeID

## 6. Sidecar Peer Discovery Tasks

### 6.1 discover-peers Task

The sidecar's `discover-peers` task (`PeerDiscoverer`) resolves peers from multiple source types and writes them to `config.toml` as `persistent-peers`.

**Task params format:**
```json
{
  "sources": [
    {"type": "ec2Tags", "region": "us-east-1", "tags": {"env": "prod"}},
    {"type": "static", "addresses": ["nodeID@host:port"]},
    {"type": "dnsEndpoints", "endpoints": ["node-0.node.ns.svc.cluster.local"]}
  ]
}
```

**Source types:**

#### EC2Tags Source (`EC2TagsSource`)
1. Creates an EC2 client for the specified region (using Pod Identity credentials)
2. Calls `DescribeInstances` with tag filters + `instance-state-name=running`
3. For each instance, gets the IP (public preferred, private fallback)
4. Queries `http://{ip}:26657/status` to get the Tendermint node ID
5. Constructs `{nodeID}@{ip}:26656` address

#### Static Source (`StaticSource`)
Returns the fixed list of addresses directly. No resolution needed.

#### DNS Endpoints Source (seictl v0.0.30+, `dnsEndpoints`)
Resolves Kubernetes DNS hostnames to peer addresses. The controller resolves label-selected SeiNode resources to their headless Service DNS names and passes them as endpoints. The sidecar queries each endpoint's RPC to get node IDs.

### 6.2 Output

All sources are deduplicated. The final peer list is written to `config.toml` via TOML merge:
```toml
[p2p]
persistent-peers = "id1@host1:26656,id2@host2:26656"
```

### 6.3 set-genesis-peers Task

For genesis ceremony nodes (`GenesisPeersSetter`):
1. Downloads `peers.json` from S3 (bucket/key/region from params)
2. Parses it as a JSON array of `"nodeID@host:port"` strings
3. Reads the local node's ID from `config/node_key.json`
4. Filters out the self-entry
5. Writes the remaining peers to `config.toml` as `persistent-peers`

## 7. State Sync Peer Selection

### 7.1 How State Sync Uses Peers

The `configure-state-sync` task (`StateSyncConfigurer`) configures Tendermint state sync, which requires:

1. **RPC servers:** At least 2 RPC endpoints that the light client queries for light blocks during verification
2. **Trust height + trust hash:** A known-good block hash at a specific height, establishing the trust anchor
3. **Trust period:** How long the validators at the trust height are considered trustworthy (default: 168h / 7 days)

### 7.2 RPC Server Selection

The state sync configurer:
1. Reads `persistent-peers` from `config.toml` (peers must already be configured by `discover-peers`)
2. Extracts up to 2 host addresses from peer strings (strips `nodeID@` prefix and `:port` suffix)
3. Uses these as RPC servers at port 26657

### 7.3 Trust Point Discovery

Two modes:

**Remote (default):** Queries `http://{rpcHost}:26657/status` for latest height, subtracts 2000 blocks as the trust height offset, then queries `http://{rpcHost}:26657/block?height={trustHeight}` for the block hash.

**Local snapshot:** When `useLocalSnapshot=true` (S3 snapshot + state sync combo), scans `data/snapshots/{height}/{format}/` for the highest snapshot height and uses that as the trust height.

### 7.4 Config Written

```toml
[statesync]
enable = true
trust-height = 12345
trust-hash = "ABCDEF..."
rpc-servers = "host1:26657,host2:26657"
trust-period = "168h0m0s"
use-local-snapshot = false
backfill-blocks = 0
```

### 7.5 What Makes a Good State Sync Peer

- Must have RPC enabled and reachable on port 26657
- Must be caught up (latest block height is used to compute trust point)
- Must serve light blocks (for light client verification during state sync)
- Must have snapshot serving enabled (`snapshot-interval > 0`) if using P2P state sync
- Should be stable (long-running, not behind on blocks)
- At least 2 peers needed for RPC server redundancy

## 8. Controller Peer Resolution

### 8.1 CRD Peer Source Types

The `SeiNode.Spec.Peers` field is a list of `PeerSource`, each a union type with exactly one of:

- **`ec2Tags`**: `{region, tags}` -- passed directly to sidecar as EC2Tags source
- **`static`**: `{addresses}` -- passed directly as Static source
- **`label`**: `{selector, namespace?}` -- resolved by the controller (not the sidecar)

### 8.2 Label Peer Resolution (`reconcilePeers`)

The node controller's `reconcilePeers()` method:
1. Iterates `node.Spec.Peers` for label sources
2. For each label source, lists `SeiNode` resources matching the label selector (in the specified or same namespace)
3. Excludes the discovering node itself
4. Constructs stable headless Service DNS names: `{name}-0.{name}.{namespace}.svc.cluster.local`
5. Sorts and deduplicates
6. Writes to `node.Status.ResolvedPeers` via optimistic-lock status patch

This runs on every reconcile, so the resolved peer list stays current as nodes are added/removed.

### 8.3 Plan Building (`discoverPeersParams`)

When building the task plan, `discoverPeersParams()` converts CRD peer sources to sidecar task params:

- `ec2Tags` -> `PeerSourceParam{Type: "ec2Tags", Region, Tags}`
- `static` -> `PeerSourceParam{Type: "static", Addresses}`
- `label` -> `PeerSourceParam{Type: "dnsEndpoints", Endpoints: node.Status.ResolvedPeers}`

The label source is converted to `dnsEndpoints` type (not `label`) so the sidecar stays infrastructure-agnostic. The controller handles Kubernetes-specific resolution; the sidecar just queries DNS hostnames for node IDs.

### 8.4 Bootstrap Plan Integration

The `discover-peers` task appears in the plan progression:
- **Base plan:** `configure-genesis -> config-apply -> [discover-peers] -> [configure-state-sync] -> config-validate -> mark-ready`
- **Bootstrap plan:** Same progression runs twice -- once on the bootstrap Job pod, once on the production StatefulSet pod (post-bootstrap)
- **Genesis ceremony:** Uses `set-genesis-peers` instead of `discover-peers`

## 9. Peer Address Format

All peer addresses follow the Tendermint format: `{nodeID}@{host}:{port}`

- `nodeID`: 40-character lowercase hex string (20 bytes from ed25519 pubkey)
- `host`: IP address or DNS hostname
- `port`: P2P port, default 26656

Example: `a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2@10.0.1.5:26656`

The port 26656 is hardcoded in the sidecar's peer discovery (`const p2pPort = "26656"`) and defined as `seiconfig.PortP2P = 26656`.

## 10. Peer Store Persistence

The peer store (`peerStore`) is backed by a key-value database (LevelDB):
- All peers are kept in memory for performance
- Changes are written back to disk (without fsync -- recent writes can be lost)
- Peer data is serialized as Protobuf (`p2pproto.PeerInfo`)
- Persisted fields: ID, addresses, last connected time, dial failure counts
- Ephemeral fields (not persisted): Persistent, Unconditional, BlockSync, Seed, Height, FixedScore, MutableScore, ConsecSuccessfulBlocks

On startup, the peer store is loaded from disk, then `configurePeers()` re-applies ephemeral flags from the current config (persistent, unconditional, blocksync).

## 11. Summary of Config Keys (config.toml)

| Config Key (TOML) | Purpose | Default (sei-config) |
|---|---|---|
| `p2p.persistent-peers` | Peers to maintain persistent connections to | "" |
| `p2p.bootstrap-peers` | Peers to seed initial discovery | "" |
| `p2p.blocksync-peers` | Peers for block sync only | "" |
| `p2p.unconditional-peer-ids` | Peer IDs that bypass connection limits | "" |
| `p2p.private-peer-ids` | Peer IDs to never gossip via PEX | "" |
| `p2p.pex` | Enable PEX reactor | true |
| `p2p.max-connections` | Max connected peers (in+out) | 100 |
| `p2p.max-incoming-connection-attempts` | Rate limit incoming per IP | 100 |
| `p2p.allow-duplicate-ip` | Allow multiple peers from same IP | false |
| `p2p.listen-address` | P2P listen address | tcp://0.0.0.0:26656 (full), tcp://127.0.0.1:26656 (base) |
| `p2p.external-address` | Address advertised to peers | "" |
| `p2p.handshake-timeout` | STS handshake timeout | 10s |
| `p2p.dial-timeout` | Dial timeout | 3s |
| `p2p.flush-throttle-timeout` | MConn flush throttle | 100ms |
| `p2p.send-rate` | Send rate (bytes/sec) | 20971520 (20 MB/s) |
| `p2p.recv-rate` | Recv rate (bytes/sec) | 20971520 (20 MB/s) |
| `p2p.max-packet-msg-payload-size` | Max packet payload | 1000000 |
| `p2p.queue-type` | Queue backend | "simple-priority" |
