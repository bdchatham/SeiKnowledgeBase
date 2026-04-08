---
title: "sei-tendermint P2P Identity, Peer Scoring, and Connection Management"
source: "github.com/sei-protocol/sei-tendermint@v0.6.4"
files_read:
  - crypto/ed25519/ed25519.go
  - crypto/crypto.go
  - types/node_id.go
  - types/node_key.go
  - types/node_info.go
  - internal/p2p/address.go
  - internal/p2p/types.go
  - internal/p2p/errors.go
  - internal/p2p/peermanager.go (complete, 1613 lines)
  - internal/p2p/peermanager_scoring_test.go
  - internal/p2p/conn/secret_connection.go
  - internal/p2p/conn/connection.go
  - internal/p2p/transport.go
  - internal/p2p/transport_mconn.go
  - internal/p2p/router.go (complete, 832 lines)
  - internal/p2p/pex/reactor.go (complete, 436 lines)
  - internal/p2p/conn_tracker.go
  - internal/p2p/metrics.go
  - node/node.go
  - node/setup.go (createPeerManager, createRouter, makeNodeInfo)
  - proto/tendermint/p2p/types.proto
scope: "Complete analysis of P2P identity derivation, STS handshake, peer scoring algorithm, peer manager state machine, PEX gossip, connection lifecycle, and Sei-specific extensions"
date: "2026-04-07"
---

# sei-tendermint P2P Identity, Peer Scoring, and Connection Management

## 1. Identity System

### 1.1 Key Generation and NodeID Derivation

The identity system is built on Ed25519 keys using the `curve25519-voi` library (with ZIP-215 verification semantics).

**Key generation chain:**

```
Ed25519 private key (64 bytes) = [32-byte seed | 32-byte compressed public key]
  -> PubKey (32 bytes) = privKey[32:64]
    -> Address (20 bytes) = SHA-256(PubKey)[:20]   (truncated hash)
      -> NodeID (string) = hex.Encode(Address)      (40 lowercase hex chars)
```

**Concrete code path:**

1. `ed25519.GenPrivKey()` -- generates 64-byte private key via `ed25519.GenerateKey(crypto/rand.Reader)`
2. `privKey.PubKey()` -- extracts bytes [32:64] as the 32-byte Ed25519 public key
3. `pubKey.Address()` -- calls `crypto.AddressHash(pubKey)` which computes `SHA-256(pubKey)[:20]`
4. `types.NodeIDFromPubKey(pubKey)` -- calls `hex.EncodeToString(pubKey.Address())` producing 40-char lowercase hex

**Source:** `crypto/ed25519/ed25519.go:165-169`, `crypto/crypto.go:27-29`, `types/node_id.go:38-39`

**NodeID validation rules:**
- Must be exactly 40 characters (2 * `AddressSize` where `AddressSize = 20`)
- Must match regex `^[0-9a-f]{40}$` (lowercase hex only)
- Empty string is rejected

### 1.2 NodeKey (Persistent Identity)

The `NodeKey` struct holds the node's persistent identity on disk:

```go
type NodeKey struct {
    ID      NodeID         // hex-encoded address
    PrivKey crypto.PrivKey // Ed25519 private key
}
```

**Persistence:** Stored as JSON in `node_key.json` at `cfg.NodeKeyFile()`. On startup:
1. `LoadOrGenNodeKey(filePath)` checks if the file exists
2. If yes: loads and unmarshals JSON, recomputes `ID` from the public key (ignoring any stored ID)
3. If no: calls `GenNodeKey()` which generates a fresh Ed25519 keypair, then saves it

**Critical detail:** The NodeID is always recomputed from the private key on load (`nodeKey.ID = NodeIDFromPubKey(nodeKey.PubKey())`), so the stored ID field is effectively a cache.

**Source:** `types/node_key.go:70-110`

### 1.3 NodeAddress Format

Peer addresses follow a URL-like format: `<nodeID>@<hostname>:<port>` with an optional protocol prefix.

```go
type NodeAddress struct {
    NodeID   types.NodeID
    Protocol Protocol  // default: "mconn"
    Hostname string
    Port     uint16
    Path     string
}
```

The default protocol is `MConnProtocol = "mconn"`. Parsing normalizes to lowercase. The `Resolve()` method performs DNS lookup to convert hostname to IP endpoints.

**Source:** `internal/p2p/address.go:34-180`

---

## 2. Connection Establishment (Complete Walkthrough)

### 2.1 Outbound Connection (Node A dials Node B)

**Step 1: Peer Selection** (`PeerManager.DialNext()` / `TryDialNext()`)

The peer manager selects the next peer to dial from a ranked list:
- Iterates `store.Ranked()` (peers sorted by score, highest first)
- Skips peers that are already `dialing` or `connected`
- Checks retry delay: `time.Since(addressInfo.LastDialFailure) < retryDelay(failures, persistent)`
- If at `MaxConnected`, looks for an upgrade candidate (lower-scored connected peer to evict)
- Marks peer as `dialing[peerID] = true`

**Step 2: DNS Resolution** (`Router.dialPeer()`)

```
NodeAddress -> address.Resolve(ctx) -> []Endpoint
```

DNS hostname is resolved to IP addresses via `net.DefaultResolver.LookupIP`. Each IP becomes an Endpoint. If resolution fails, the peer is marked as private (won't be gossiped).

**Step 3: TCP Connection** (`MConnTransport.Dial()`)

```go
dialer := net.Dialer{}
tcpConn, err := dialer.DialContext(ctx, "tcp", endpoint.Addr.String())
```

Returns a raw `net.Conn` wrapped in an `mConnConnection`.

**Step 4: STS Handshake** (`mConnConnection.handshake()`)

4a. **Secret Connection** (`conn.MakeSecretConnection(tcpConn, privKey)`)

See Section 3 for full STS protocol details. Produces an authenticated, encrypted `SecretConnection`.

4b. **NodeInfo Exchange** (over the encrypted SecretConnection)

Both sides simultaneously send/receive their `NodeInfo` (protobuf-encoded) via `protoio.NewDelimitedWriter/Reader`.

4c. **MConnection Creation**

A multiplexed `MConnection` is created over the `SecretConnection` with configured channel descriptors.

**Step 5: Peer Authentication** (`Router.handshakePeer()`)

```go
// Verify the peer's public key matches their claimed NodeID
if types.NodeIDFromPubKey(peerKey) != peerInfo.NodeID {
    return error // reject
}
```

Additional checks:
- `peerInfo.Validate()` -- structural validation
- `peerInfo.Network != nodeInfo.Network` -- network mismatch causes peer deletion from store
- `nodeInfo.CompatibleWith(peerInfo)` -- block protocol version match, at least one common channel
- Optional `FilterPeerByIP` and `FilterPeerByID` hooks

**Step 6: Peer Manager Registration** (`PeerManager.Dialed()`)

- Removes from `dialing` map
- Checks for self-connection, duplicate connection
- If upgrading: finds lowest-scored connected peer to evict
- Sets `connected[peerID] = true`, updates `LastConnected` timestamp
- Resets dial failures for the successful address
- Wakes evict waker

**Step 7: Route Peer** (`Router.routePeer()`)

- Calls `peerManager.Ready(peerID, channels)` which sets `ready[peerID] = true` and broadcasts `PeerStatusUp`
- Spawns two goroutines: `sendPeer()` and `receivePeer()`
- Creates a per-peer `Queue` for outbound messages

### 2.2 Inbound Connection (Node B receives from Node A)

1. `Transport.Accept()` yields raw TCP connection
2. Rate limiting via `connTracker.AddConn(incomingAddr)` (per-IP limits)
3. IP filtering via `FilterPeerByIP` hook
4. Same STS handshake + NodeInfo exchange
5. `PeerManager.Accepted(peerID)` -- same upgrade logic but can't take an address (inbound port differs from listening port)
6. If unconditional peer: bypasses `MaxConnected` limit check entirely
7. Same `routePeer()` flow

**Source:** `internal/p2p/router.go:400-693`

---

## 3. STS (Station-to-Station) Handshake Protocol

The handshake establishes an authenticated, encrypted channel. It implements a variant of the Station-to-Station protocol with X25519 key exchange and ChaCha20-Poly1305 encryption.

### 3.1 Protocol Steps

```
Node A                                          Node B
------                                          ------
1. Generate ephemeral X25519 keypair
   (locEphPub, locEphPriv)
                                                1. Generate ephemeral X25519 keypair
                                                   (locEphPub, locEphPriv)

2. Send locEphPub -----> (simultaneously) <----- 2. Send locEphPub
   Receive remEphPub                               Receive remEphPub

3. Sort ephemeral pubkeys lexicographically:
   loEphPub = min(locEphPub, remEphPub)
   hiEphPub = max(locEphPub, remEphPub)
   locIsLeast = (locEphPub == loEphPub)

4. Build Merlin transcript:
   transcript.AppendMessage("EPHEMERAL_LOWER_PUBLIC_KEY", loEphPub)
   transcript.AppendMessage("EPHEMERAL_UPPER_PUBLIC_KEY", hiEphPub)

5. Compute DH secret:
   dhSecret = X25519(locEphPriv, remEphPub)

6. Append DH secret to transcript:
   transcript.AppendMessage("DH_SECRET", dhSecret)

7. Derive send/recv keys via HKDF-SHA256:
   hkdf = HKDF(SHA256, dhSecret, salt=nil, info="TENDERMINT_SECRET_CONNECTION_KEY_AND_CHALLENGE_GEN")
   [key1 | key2 | challenge_material] = hkdf.Read(2*32 + 32)
   
   if locIsLeast:
     recvSecret = key1, sendSecret = key2
   else:
     sendSecret = key1, recvSecret = key2

8. Extract challenge from transcript:
   challenge[32] = transcript.ExtractBytes("SECRET_CONNECTION_MAC")

9. Create AEAD ciphers:
   sendAead = ChaCha20-Poly1305(sendSecret)
   recvAead = ChaCha20-Poly1305(recvSecret)

--- Encrypted channel established ---

10. Sign challenge with persistent Ed25519 key:
    signature = Ed25519.Sign(privKey, challenge)

11. Exchange (encrypted) auth signatures:
    Send {PubKey, Signature} -----> (simultaneously) <----- Send {PubKey, Signature}
    Receive {remPubKey, remSig}                              Receive {remPubKey, remSig}

12. Verify remote signature:
    remPubKey.VerifySignature(challenge, remSig) must be true
    remPubKey must be Ed25519 type
```

### 3.2 Cryptographic Primitives

| Component | Algorithm | Library |
|-----------|-----------|---------|
| Ephemeral key exchange | X25519 (Curve25519 ECDH) | `golang.org/x/crypto/curve25519` |
| Ephemeral key generation | NaCl box.GenerateKey | `golang.org/x/crypto/nacl/box` |
| Key derivation | HKDF-SHA256 | `golang.org/x/crypto/hkdf` |
| Transcript hashing | Merlin transcripts | `oasisprotocol/curve25519-voi/primitives/merlin` |
| Symmetric encryption | ChaCha20-Poly1305 (IETF) | `golang.org/x/crypto/chacha20poly1305` |
| Authentication signing | Ed25519 | `oasisprotocol/curve25519-voi/primitives/ed25519` |
| Signature verification | Ed25519 with ZIP-215 | Same, with caching verifier (LRU 4096) |

### 3.3 Encrypted Transport Details

After handshake, all data is framed and encrypted:

- **Frame size:** 4 bytes (data length, little-endian) + up to 1024 bytes payload = 1028 bytes max per frame
- **Encrypted frame:** 1028 + 16 (Poly1305 tag) = 1044 bytes on wire
- **Nonce:** 12-byte counter (first 4 bytes zero, last 8 bytes little-endian uint64, incremented per frame)
- **Nonce overflow:** panics at `MaxUint64` (terminates session to prevent nonce reuse)

**Source:** `internal/p2p/conn/secret_connection.go:32-465`

---

## 4. Peer Scoring System

### 4.1 Score Types and Constants

```go
type PeerScore uint8  // 0-255

PeerScoreUnconditional    = 255  // unconditional peers (always connected)
PeerScorePersistent       = 254  // persistent peers from config
MaxPeerScoreNotPersistent = 253  // hard cap for regular peers
DefaultMutableScore       = 243  // starting score for new peers (253 - 10)
```

### 4.2 Score Calculation (`peerInfo.Score()`)

The scoring function computes a composite score with the following components:

```go
func (p *peerInfo) Score() PeerScore {
    // 1. Fixed scores (testing/override)
    if p.FixedScore > 0 { return p.FixedScore }
    
    // 2. Unconditional peers always get max score
    if p.Unconditional { return 255 }
    
    // 3. Base score
    score := int64(p.MutableScore)  // starts at 243 for new peers
    if p.Persistent || p.BlockSync {
        score = int64(PeerScorePersistent)  // 254
    }
    
    // 4. Block sync bonus: +1 per 5 consecutive successful blocks
    score += p.ConsecSuccessfulBlocks / 5
    
    // 5. Dial failure penalty (with exponential time decay)
    for _, addr := range p.AddressInfo {
        failureScore := float64(addr.DialFailures) * math.Exp(-0.1 * hoursSinceLastFailure)
        score -= int64(failureScore)
    }
    
    // 6. Disconnection penalty (with exponential time decay)
    decayFactor := math.Exp(-0.1 * hoursSinceLastConnect)
    effectiveDisconnections := int64(float64(p.NumOfDisconnections) * decayFactor)
    score -= effectiveDisconnections / 3
    
    // 7. Cap for non-persistent peers
    if !p.Persistent && score > 253 { score = 253 }
    
    // 8. Floor at zero
    if score <= 0 { return 0 }
    
    return PeerScore(score)
}
```

### 4.3 Score Component Details

| Component | Effect | Details |
|-----------|--------|---------|
| **MutableScore** | Base score | Starts at 243; +1 per `PeerStatusGood` event, -1 per `PeerStatusBad` event |
| **Persistent/BlockSync flag** | Overrides base to 254 | Config-driven; these peers are always high-priority |
| **Unconditional flag** | Fixed at 255 | Bypasses all scoring; always connected regardless of limits |
| **ConsecSuccessfulBlocks** | +1 per 5 blocks | Rewards peers that serve blocks reliably; reset on disconnect or dial failure |
| **Dial failures** | Penalty with decay | `failures * e^(-0.1 * hours)` -- recent failures hurt more, old ones decay away |
| **Disconnections** | Penalty with decay | `disconnections * e^(-0.1 * hours) / 3` -- moderate penalty, heavier with recency |
| **Non-persistent cap** | Max 253 | Prevents regular peers from reaching persistent/unconditional tier |
| **Floor** | Min 0 | Score can't go negative |

### 4.4 Score Mutation Points

| Event | Score Effect | Code Location |
|-------|-------------|---------------|
| `processPeerEvent(PeerStatusGood)` | `MutableScore++` | peermanager.go:1027 |
| `processPeerEvent(PeerStatusBad)` | `MutableScore--` | peermanager.go:1024 |
| `IncrementBlockSyncs(peerID)` | `ConsecSuccessfulBlocks++` (affects score via /5) | peermanager.go:1605-1613 |
| `DialFailed()` | `addr.DialFailures++`, `ConsecSuccessfulBlocks = 0` | peermanager.go:615-618 |
| `Disconnected()` | `NumOfDisconnections++`, `ConsecSuccessfulBlocks = 0` | peermanager.go:866-876 |
| `Ban()` | `FixedScore = 0`, `MutableScore = 0`, `ConsecSuccessfulBlocks = 0` | peermanager.go:1506-1515 |
| `Dialed()` (success) | `addr.DialFailures = 0` | peermanager.go:685 |
| `Accepted()` | All `addr.DialFailures = 0` (reset past transgressions) | peermanager.go:749-751 |
| Channel error (non-fatal, not at capacity) | `PeerStatusBad` -> `MutableScore--` | router.go:270-273 |
| Channel error (fatal or at capacity) | `Errored()` -> scheduled for eviction | router.go:267-269 |

### 4.5 Ranking and Cache

Peers are ranked by `peerStore.Ranked()` which sorts all peers by `Score()` descending. The sorted list is cached (`s.ranked`) and invalidated (set to nil) whenever:
- A peer's score changes
- A peer is added or deleted
- A peer disconnects
- A dial failure occurs
- A `PeerStatusGood/Bad` event is processed

Ties in score are broken by NodeID lexicographic order (for deterministic behavior in tests).

**Source:** `internal/p2p/peermanager.go:1345-1368`

---

## 5. Peer Eviction

### 5.1 Eviction Triggers

Peers are scheduled for eviction (`m.evict[peerID] = error`) in three scenarios:

1. **Upgrade eviction:** When a higher-scored peer connects (via `Dialed()` or `Accepted()`) and we're at `MaxConnected`, the lowest-scored connected peer is evicted
2. **Error eviction:** `PeerManager.Errored(peerID, err)` schedules immediate eviction for peers with errors (called when channel errors are fatal or when at max capacity)
3. **Over-capacity:** If somehow above `MaxConnected` (shouldn't normally happen), `TryEvictNext()` picks the lowest-ranked connected peer

### 5.2 Eviction Flow

```
evict[peerID] set ──> evictWaker.Wake()
  ──> EvictNext() / TryEvictNext()
    ──> marks evicting[peerID] = true
      ──> Router cancels peer context (s.cancel())
        ──> connection goroutines exit
          ──> Disconnected(peerID)
            ──> clears connected, upgrading, evict, evicting, ready
              ──> broadcasts PeerStatusDown
                ──> dialWaker.Wake() (allow re-dial)
```

### 5.3 Eviction Protection

- **Unconditional peers** are never evicted (score 255, always above any upgrade candidate)
- **Persistent peers** have score 254, so they're only evicted by other persistent/unconditional peers
- Peers currently being evicted (`evicting[peerID]`) are not selected again
- Peers with pending eviction (`evict[peerID]`) are not selected as upgrade candidates

### 5.4 Ban Mechanism

`BanPeer(id)` sets `FixedScore = 0`, `MutableScore = 0`, `ConsecSuccessfulBlocks = 0`. However, persistent and block-sync peers **cannot be banned** -- the function is a no-op with a log message.

**Source:** `internal/p2p/peermanager.go:1203-1210, 1506-1515`

---

## 6. Peer Store (Persistence)

### 6.1 Storage Backend

Peers are stored in a key-value database (typically GoLevelDB) with protobuf serialization:

```
Key:   orderedcode.Append(nil, prefixPeerInfo=1, string(nodeID))
Value: proto.Marshal(p2pproto.PeerInfo)
```

### 6.2 Persisted Fields

The protobuf schema (`proto/tendermint/p2p/types.proto`) stores:

```protobuf
message PeerInfo {
    string id = 1;
    repeated PeerAddressInfo address_info = 2;
    google.protobuf.Timestamp last_connected = 3;
}

message PeerAddressInfo {
    string address = 1;           // URL string: "nodeID@host:port"
    Timestamp last_dial_success = 2;
    Timestamp last_dial_failure = 3;
    uint32 dial_failures = 4;
}
```

### 6.3 Ephemeral Fields (NOT Persisted)

These fields exist in-memory only and are reconfigured on startup:

| Field | Type | Purpose |
|-------|------|---------|
| `Persistent` | bool | Set from `PersistentPeers` config |
| `Unconditional` | bool | Set from `UnconditionalPeers` config |
| `BlockSync` | bool | Set from `BlockSyncPeers` config |
| `Seed` | bool | Seed node flag |
| `Height` | int64 | Peer's reported height |
| `FixedScore` | PeerScore | Override score (testing) |
| `MutableScore` | PeerScore | Dynamic score component |
| `ConsecSuccessfulBlocks` | int64 | Consecutive block sync successes |
| `NumOfDisconnections` | int64 | Total disconnection count |

**Critical implication:** `MutableScore`, `NumOfDisconnections`, and `ConsecSuccessfulBlocks` are lost on restart. Every peer starts fresh with `DefaultMutableScore = 243` after restart. Only dial failure counts and timestamps survive.

### 6.4 In-Memory Architecture

The entire peer set is loaded into memory on initialization. All operations work on the in-memory map, with writes going to the database for durability (without fsync -- recent writes may be lost on crash).

```go
type peerStore struct {
    db      dbm.DB
    peers   map[types.NodeID]*peerInfo  // complete in-memory set
    ranked  []*peerInfo                  // sorted cache, nil = invalid
    metrics *Metrics
}
```

**Source:** `internal/p2p/peermanager.go:1212-1598`

---

## 7. PEX (Peer Exchange) Protocol

### 7.1 Protocol Overview

PEX runs on channel ID `0x00` and has two message types:

- **PexRequest:** "Give me peer addresses"
- **PexResponse:** "Here are up to 100 peer addresses"

### 7.2 PEX Reactor Flow

1. **On peer connect (`PeerStatusUp`):** Add peer to `availablePeers` set
2. **On peer disconnect (`PeerStatusDown`):** Remove from `availablePeers` and `requestsSent`
3. **Periodic polling:** Select a random available peer, send `PexRequest`, move to `requestsSent`
4. **On response:** Parse addresses, call `peerManager.Add()` for each, adjust polling interval

### 7.3 Rate Limiting

- **Outbound requests:** Adaptive polling interval based on network knowledge
  - Starts at `minReceiveRequestInterval = 100ms` (bootstrap)
  - Scales based on `totalPeers / newPeersAdded` ratio
  - At 95% peer store capacity: backs off to `fullCapacityInterval = 10 minutes`
  - No available peers: `noAvailablePeersWaitPeriod = 1 second` with exponential backoff
  
- **Inbound requests:** `minReceiveRequestInterval = 100ms` between requests from the same peer

### 7.4 Address Selection for Responses

`peerManager.Advertise(peerID, limit=100)` returns:
1. Self-address first (if configured with hostname and port)
2. Remaining slots filled from highest-ranked peers' addresses
3. Private peers (those in `PrivatePeers` set) are excluded

### 7.5 Self-Remediation

If all peers disconnect and no available peers remain for longer than `P2pNoPeersRestarWindowSeconds`, the PEX reactor sends a signal to restart the entire router (node-level self-remediation).

**Source:** `internal/p2p/pex/reactor.go:1-436`

---

## 8. Connection Limits and Enforcement

### 8.1 Configuration Defaults (from `createPeerManager`)

```go
MaxConnected:           64  (configurable via P2P.MaxConnections)
MaxConnectedUpgrade:    4   (hardcoded)
MaxPeers:               4 + 2*64 = 132  (upgrade + 2*connected)
MinRetryTime:           250ms
MaxRetryTime:           2 minutes
MaxRetryTimePersistent: 2 minutes
RetryTimeJitter:        5 seconds
```

### 8.2 Connection Slot Accounting

```
Total dial budget = MaxConnected + MaxConnectedUpgrade = 68

NumConnected() counts only non-unconditional peers.
Unconditional peers bypass ALL limits.
```

**Dialing throttle:** The router sleeps 250ms-3s (random) between dial attempts to avoid network spam. Dials run in `runtime.NumCPU()` parallel goroutines.

### 8.3 Incoming Connection Rate Limiting

The `connTracker` limits per-IP connections:
- `MaxIncomingConnectionAttempts = 100` (per IP)
- `IncomingConnectionWindow = 100ms` (minimum interval between connection attempts from same IP)
- Transport-level `MaxAcceptedConnections` limits total concurrent accepted connections

### 8.4 Peer Store Pruning

When `store.Size() > MaxPeers`, `prunePeers()` deletes the lowest-scored unconnected, non-dialing peers to make room.

**Source:** `internal/p2p/peermanager.go:102-175, 433-455`, `internal/p2p/conn_tracker.go`

---

## 9. Persistent Peers vs Regular Peers vs Unconditional Peers

### 9.1 Persistent Peers

- **Score:** Base of 254 (overrides MutableScore)
- **Retry:** Uses `MaxRetryTimePersistent` (default 2 min, can be different from regular)
- **Eviction:** Only evicted by higher-scored peers (other persistent or unconditional)
- **Banning:** Cannot be banned (Ban() is a no-op)
- **Pruning:** Higher score means they survive pruning
- **Config:** Set via `P2P.PersistentPeers` in config.toml

### 9.2 Unconditional Peers

- **Score:** Fixed at 255 (maximum possible)
- **Connection limits:** Completely bypassed -- `NumConnected()` excludes unconditional peers, and `Accepted()` skips the `MaxConnected` check for them
- **Eviction:** Never evicted (no peer can score higher)
- **Store size:** Excluded from `peerStore.Size()` count, so they don't count toward MaxPeers
- **Config:** Set via `P2P.UnconditionalPeerIDs` in config.toml

### 9.3 BlockSync Peers (Sei-specific)

- **Score:** Same as persistent (254 base)
- **Behavior:** Identical to persistent peers in scoring and eviction
- **Banning:** Cannot be banned
- **Additional:** `peerManager.GetBlockSyncPeers()` returns the set for reactor use
- **Config:** Set via `P2P.BlockSyncPeers` in config.toml (Sei extension)

### 9.4 Regular Peers

- **Score:** Starts at `DefaultMutableScore = 243`, fluctuates based on behavior
- **Cap:** Cannot exceed `MaxPeerScoreNotPersistent = 253`
- **Eviction:** First to be evicted when connection slots are needed
- **Banning:** Can be banned (score forced to 0)

---

## 10. Retry and Backoff Logic

### 10.1 Exponential Backoff

```go
func retryDelay(failures uint32, persistent bool) time.Duration {
    if failures == 0 { return 0 }                    // immediate retry on first attempt
    if MinRetryTime == 0 { return retryNever }        // retries disabled
    
    maxDelay := MaxRetryTime                          // 2 minutes
    if persistent && MaxRetryTimePersistent > 0 {
        maxDelay = MaxRetryTimePersistent              // 2 minutes (same by default)
    }
    
    delay := MinRetryTime * 2^failures                // 250ms, 500ms, 1s, 2s, 4s...
    delay += random(0, RetryTimeJitter)               // + 0-5s jitter
    
    if delay > maxDelay { delay = maxDelay }          // cap at 2 minutes
    return delay
}
```

### 10.2 Maximum Failures Handling

When `retryDelay` reaches `MaxRetryTime`, the peer is **deleted from the store entirely** and a `DialFailuresError` is returned. This is a permanent eviction from the address book -- the peer must be re-added (e.g., via PEX or config).

### 10.3 Retry Scheduling

After a dial failure, a goroutine is spawned that sleeps for the retry delay and then wakes the dial waker, causing `DialNext()` to reconsider this peer.

**Source:** `internal/p2p/peermanager.go:1175-1201, 594-646`

---

## 11. Peer Lifecycle State Machine

```
                                 ┌──────────────────┐
                                 │                   │
                                 │  Unknown/Stored   │
                                 │  (in peer store)  │
                                 │                   │
                                 └────────┬──────────┘
                                          │
                          ┌───────────────┴───────────────┐
                          │                               │
                   DialNext()                       Accept() from
                          │                          transport
                          ▼                               │
                 ┌────────────────┐                       │
                 │   Dialing      │                       │
                 │ dialing[id]=T  │                       │
                 └───────┬────────┘                       │
                         │                                │
              ┌──────────┴──────────┐                     │
              │                     │                     │
        DialFailed()          Dialed()              Accepted()
              │                     │                     │
              ▼                     ▼                     ▼
     ┌──────────────┐     ┌──────────────────┐  ┌──────────────────┐
     │ Retry/Delete │     │   Connected      │  │   Connected      │
     │ (back to     │     │ connected[id]=T  │  │ connected[id]=T  │
     │  Unknown)    │     │                  │  │                  │
     └──────────────┘     └────────┬─────────┘  └────────┬─────────┘
                                   │                      │
                                   └──────────┬───────────┘
                                              │
                                         Ready()
                                              │
                                              ▼
                                    ┌──────────────────┐
                                    │     Ready        │
                                    │  ready[id]=T     │
                                    │  PeerStatusUp    │
                                    │  broadcast       │
                                    └────────┬─────────┘
                                             │
                              ┌──────────────┴──────────────┐
                              │                             │
                      Disconnected()                   Errored()
                              │                             │
                              │                    evict[id] = err
                              │                    evictWaker.Wake()
                              │                             │
                              │                      EvictNext()
                              │                    evicting[id]=T
                              │                    Router cancels ctx
                              │                             │
                              ▼                             ▼
                    ┌──────────────────┐          ┌──────────────────┐
                    │  Disconnected    │          │  Disconnected    │
                    │  PeerStatusDown  │◄─────────│  (via eviction)  │
                    │  broadcast       │          │                  │
                    │  dialWaker.Wake()│          │                  │
                    └──────────────────┘          └──────────────────┘
```

### State Maps

| Map | Meaning | Set By | Cleared By |
|-----|---------|--------|------------|
| `dialing[id]` | Outbound dial in progress | `TryDialNext()` | `Dialed()`, `DialFailed()` |
| `connected[id]` | TCP connection established | `Dialed()`, `Accepted()` | `Disconnected()` |
| `ready[id]` | Handshake complete, routing active | `Ready()` | `Disconnected()` |
| `upgrading[from]=to` | Upgrade in progress (from will be evicted for to) | `TryDialNext()` | `Dialed()`, `DialFailed()`, `Disconnected()` |
| `evict[id]` | Scheduled for eviction | `Dialed()`, `Accepted()`, `Errored()` | `TryEvictNext()`, `Disconnected()` |
| `evicting[id]` | Eviction in progress | `TryEvictNext()` | `Disconnected()` |

---

## 12. MConnection (Multiplexed Connection)

### 12.1 Channel System

Each MConnection multiplexes messages across multiple channels. Each channel has:
- `ID` (uint16, but restricted to uint8 for MConn)
- `Priority` (int, higher = more bandwidth)
- `SendQueueCapacity` and `RecvBufferCapacity`
- `MessageType` (protobuf message for deserialization)

### 12.2 Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `MaxPacketMsgPayloadSize` | 1400 bytes | Max payload per packet |
| `FlushThrottle` | 100ms | Throttle interval for flushing writes |
| `SendRate` | 512,000 B/s | Outbound bandwidth limit |
| `RecvRate` | 512,000 B/s | Inbound bandwidth limit |
| `PingInterval` | 60s | Keep-alive ping frequency |
| `PongTimeout` | 90s | Disconnect if no pong received |

### 12.3 Flow Control

MConnection implements bandwidth-limited multiplexing:
- Messages are split into packets of `MaxPacketMsgPayloadSize`
- `sendMonitor` and `recvMonitor` enforce rate limits
- Higher-priority channels get proportionally more bandwidth
- `numBatchPacketMsgs = 10` packets are batched per send cycle

**Source:** `internal/p2p/conn/connection.go:29-199`

---

## 13. Router Architecture

### 13.1 Core Goroutines (spawned on Start)

1. **`transport.Run`** -- listens for TCP connections
2. **`dialPeers`** -- loops calling `peerManager.DialNext()`, spawns worker pool for dialing
3. **`acceptPeers`** -- loops calling `transport.Accept()`, spawns goroutine per inbound connection
4. **`evictPeers`** -- loops calling `peerManager.EvictNext()`, cancels peer context

### 13.2 Per-Peer Goroutines

- **`sendPeer`** -- dequeues from peer's outbound `Queue`, serializes, sends via MConnection
- **`receivePeer`** -- reads from MConnection, deserializes, enqueues to channel queue

### 13.3 Per-Channel Goroutines

- **`routeChannel`** -- dequeues from channel's outbound queue, routes to target peer's queue
- **Error handler** -- reads from channel error queue, reports to peer manager

### 13.4 Message Queue System

Two levels of queuing:
1. **Channel queues** (`channelQueues`): inbound messages from all peers for a given channel (consumed by reactor)
2. **Peer queues** (`peerState.queue`): outbound messages for a specific peer across all channels (consumed by sendPeer)

Queues use priority-based insertion and can drop lower-priority messages under contention.

**Source:** `internal/p2p/router.go:98-832`

---

## 14. Implications for Controller Peer Management

### 14.1 What the Controller Needs to Provide

For the node to connect to peers, the controller needs to provide peer addresses in the format:
```
<40-char-hex-nodeID>@<hostname-or-ip>:<port>
```

These are configured via:
- `persistent-peers` in config.toml (for persistent peers)
- `bootstrap-peers` in config.toml (for initial bootstrap, not persistent)
- `unconditional-peer-ids` in config.toml (just IDs, no addresses -- these peers must also appear in persistent or bootstrap peers)
- `blocksync-peers` in config.toml (Sei extension, treated like persistent)

### 14.2 Controller Peer Resolution Alignment

The controller's peer resolution must produce addresses containing **valid NodeIDs**. The NodeID is derived from the node's Ed25519 public key and cannot be fabricated -- it must come from the actual `node_key.json` of the target peer.

For peers within a SeiNodeGroup, the controller has access to these keys (or can read them from the node's PVC). For external peers, the full `nodeID@host:port` string must be known.

### 14.3 Network Isolation

Peers are validated during handshake for `Network` (chain ID) match. Peers from different chains are **deleted from the peer store** immediately. This provides natural isolation between chains.

### 14.4 Score Volatility on Restart

Since `MutableScore`, `NumOfDisconnections`, and `ConsecSuccessfulBlocks` are ephemeral (not persisted), all peers restart at `DefaultMutableScore = 243` after node restart. Only `DialFailures` and timestamps survive. This means the scoring system provides short-term optimization within a node's uptime but does not carry long-term reputation.

### 14.5 Unconditional Peers for Intra-Group Connectivity

For nodes within the same SeiNodeGroup that must always maintain connectivity (e.g., validators and their sentries), `unconditional-peer-ids` is the strongest guarantee. These peers:
- Bypass all connection limits
- Are excluded from connection counting
- Cannot be evicted
- Always score 255

### 14.6 PEX and Peer Discovery

PEX provides automatic peer discovery but is rate-limited and adaptive. For controlled environments, it may be desirable to disable PEX (`pex = false` in config.toml) and rely entirely on controller-managed `persistent-peers` and `bootstrap-peers` for deterministic peer topology.
