---
title: Validator Identity, Key Management, and Remote Signing in Sei
category: consensus
sourced_from:
  - repo: sei-protocol/sei-tendermint@v0.6.4
    files:
      - privval/file.go
      - privval/signer_client.go
      - privval/signer_listener_endpoint.go
      - privval/signer_dialer_endpoint.go
      - privval/signer_server.go
      - privval/signer_endpoint.go
      - privval/signer_requestHandler.go
      - privval/secret_connection.go
      - privval/socket_dialers.go
      - privval/socket_listeners.go
      - privval/retry_signer_client.go
      - privval/msgs.go
      - privval/errors.go
      - privval/utils.go
      - privval/doc.go
      - privval/grpc/client.go
      - privval/grpc/server.go
      - privval/grpc/util.go
      - types/priv_validator.go
      - types/node_key.go
      - node/node.go
      - node/setup.go
      - config/config.go
      - config/toml.go
      - crypto/crypto.go
      - crypto/ed25519/ed25519.go
      - internal/consensus/state.go
      - proto/tendermint/privval/types.proto
      - proto/tendermint/privval/service.proto
  - repo: sei-protocol/sei-cosmos@v0.3.66
    files:
      - crypto/keyring/keyring.go
      - x/staking/types/tx.pb.go
      - x/staking/types/staking.pb.go
      - x/staking/client/cli/tx.go
      - x/slashing/types/params.go
      - x/genutil/utils.go
  - repo: sei-protocol/sei-chain@v0.0.38
    files:
      - (no Sei-specific modifications to validator/signing machinery)
  - repo: sei-protocol/seictl@v0.0.23
    files:
      - sidecar/tasks/generate_identity.go
      - sidecar/tasks/generate_gentx.go
relates_to:
  - memories/networking/peer-resolution.md
  - memories/upgrades/sidecar-bootstrap-lifecycle.md
updated: "2026-04-07"
---

# Validator Identity, Key Management, and Remote Signing

This document exhaustively describes the validator identity system in sei-tendermint,
the operator keyring in sei-cosmos, and how seictl bootstraps validator nodes. It is
the reference for implementing validator mode in the k8s controller.

---

## 1. The Two Identities: Node Key vs. Validator Key

A Sei node has **two distinct cryptographic identities** that must not be confused:

### 1.1 Node Key (`node_key.json`)

**Purpose:** P2P network identity. Used for authenticated encrypted connections between peers.

**Location:** `$HOME/config/node_key.json`

**Format:**
```json
{
  "id": "<hex-encoded-20-byte-address>",
  "priv_key": {"type": "tendermint/PrivKeyEd25519", "value": "<base64-64-bytes>"}
}
```

**Key type:** Always Ed25519 (hardcoded in `GenNodeKey()`).

**Generation:** `types.GenNodeKey()` calls `ed25519.GenPrivKey()` and derives
`NodeID = NodeIDFromPubKey(privKey.PubKey())` which is the hex-encoded first 20 bytes
of SHA-256(pubkey).

**Security implications:** Compromise allows impersonation on the P2P network but does
NOT allow signing consensus votes. Can be regenerated without slashing risk, though
persistent peers must be updated.

**Relevant code:** `sei-tendermint/types/node_key.go`

### 1.2 Validator Key (`priv_validator_key.json`)

**Purpose:** Consensus signing identity. Signs votes (prevote, precommit) and proposals.
This IS the validator's identity in the consensus protocol. Compromise or misuse leads
to **double-sign slashing** (tombstoning).

**Location:** `$HOME/config/priv_validator_key.json`

**Format:**
```json
{
  "address": "<hex-encoded-20-byte-address>",
  "pub_key": {"type": "tendermint/PubKeyEd25519", "value": "<base64-32-bytes>"},
  "priv_key": {"type": "tendermint/PrivKeyEd25519", "value": "<base64-64-bytes>"}
}
```

**Key type:** Ed25519 by default, secp256k1 also supported (configurable via `keyType`
parameter in `GenFilePV`). The `address` field is `SHA256(pubkey)[:20]` -- a truncated
SHA-256 hash of the raw public key bytes.

**Ed25519 specifics:**
- Private key: 64 bytes (first 32 = seed, last 32 = compressed public key)
- Public key: 32 bytes
- Signature: 64 bytes
- Uses `curve25519-voi` implementation with ZIP-215 verification semantics
- Generated via `ed25519.GenPrivKey()` using `crypto/rand.Reader`
- Deterministic derivation possible via `GenPrivKeyFromSecret(secret)` which
  hashes the secret with SHA-256 and uses the result as seed

**File permissions:** Written with mode 0600 (owner read/write only) via atomic write
(`tempfile.WriteFileAtomic`).

**Relevant code:** `sei-tendermint/privval/file.go` (struct `FilePVKey`)

---

## 2. Validator State (`priv_validator_state.json`)

**Purpose:** Double-sign prevention. Tracks the last signed height/round/step to ensure
the validator never signs conflicting messages for the same consensus round.

**Location:** `$HOME/data/priv_validator_state.json`

**Format:**
```json
{
  "height": "12345",
  "round": 0,
  "step": 3,
  "signature": "<hex-signature>",
  "signbytes": "<hex-signbytes>"
}
```

**Fields:**
- `height` (string-encoded int64): Last signed block height
- `round` (int32): Last signed round within the height
- `step` (int8): Last signed step -- 0=none, 1=propose, 2=prevote, 3=precommit
- `signature` (bytes): The last signature produced (for crash recovery)
- `signbytes` (bytes): The bytes that were signed (for crash recovery)

**Double-sign prevention logic (`checkHRS`):**
The state enforces a strict monotonic ordering: height must never decrease, round must
never decrease within the same height, and step must never decrease within the same
height+round. If the exact same HRS is requested again:
- If signbytes match: reuse the cached signature (crash recovery)
- If only timestamp differs: reuse the last timestamp and signature
- Otherwise: return "conflicting data" error

This is **critical for safety**. If a validator signs two different votes at the same
height+round+step, it constitutes a **double sign** which results in tombstoning.

**Operational implications:**
- This file MUST be persisted across container restarts on the SAME PVC as the node data
- If lost and the node restarts, it could re-sign at the same HRS with different data
- For remote signing, the KMS maintains this state instead

**Relevant code:** `sei-tendermint/privval/file.go` (struct `FilePVLastSignState`)

---

## 3. The PrivValidator Interface

```go
type PrivValidator interface {
    GetPubKey(context.Context) (crypto.PubKey, error)
    SignVote(ctx context.Context, chainID string, vote *tmproto.Vote) error
    SignProposal(ctx context.Context, chainID string, proposal *tmproto.Proposal) error
}
```

**Implementations (with type enum for metrics):**

| Type Enum | Value | Implementation | Description |
|-----------|-------|----------------|-------------|
| `MockSignerClient` | 0x00 | `MockPV` | Testing only, no safety checks |
| `FileSignerClient` | 0x01 | `*FilePV` | File-based, production default |
| `RetrySignerClient` | 0x02 | `*RetrySignerClient` | Wraps SignerClient with retry logic |
| `SignerSocketClient` | 0x03 | `*SignerClient` | Socket-based remote signer |
| `ErrorMockSignerClient` | 0x04 | `*ErroringMockPV` | Testing only, always errors |
| `SignerGRPCClient` | 0x05 | `*grpc.SignerClient` | gRPC-based remote signer |

The consensus `State` uses a type switch on `SetPrivValidator` to determine which
implementation is in use, stored as `cs.privValidatorType` for metrics/logging.

---

## 4. FilePV: File-Based Signing (Default)

`FilePV` combines `FilePVKey` (immutable key material) and `FilePVLastSignState`
(mutable signing state) into a single validator implementation.

**Lifecycle:**
1. `GenFilePV(keyPath, statePath, keyType)` -- generates random key, returns unsaved FilePV
2. `LoadOrGenFilePV(keyPath, statePath)` -- loads existing or generates+saves new
3. `LoadFilePV(keyPath, statePath)` -- loads both key and state from disk
4. `LoadFilePVEmptyState(keyPath, statePath)` -- loads key only, empty state (for reset)

**Signing flow (`signVote`):**
1. Convert vote type to step (prevote=2, precommit=3)
2. Call `checkHRS(height, round, step)` against last sign state
3. If same HRS and same signbytes: return cached signature (crash recovery)
4. If same HRS and only timestamp differs: use last timestamp (crash recovery)
5. If same HRS and different data: return "conflicting data" error
6. Otherwise: sign with private key, persist new state, return signature

**Vote extensions:** For non-nil precommit votes, extension signatures are always
re-signed (extensions are non-deterministic). For prevotes and nil precommits,
extensions must be empty.

---

## 5. Remote Signing Architecture

### 5.1 Overview

Remote signing separates the signing key from the validator node. The node runs in
"listener" mode, and an external Key Management Service (KMS) dials into the node to
provide signing services. This is the production-recommended configuration for
validators with significant stake.

### 5.2 Configuration (`config.toml`)

```toml
[priv-validator]
# Path to key and state files (used only for file-based signing)
key-file = "config/priv_validator_key.json"
state-file = "data/priv_validator_state.json"

# TCP or UNIX socket address for the node to listen on
# for connections from an external PrivValidator process.
# When prefixed with "grpc" it uses gRPC instead of raw socket.
# Empty string = use file-based signing (default)
laddr = ""

# TLS certificates for secure gRPC connections
client-certificate-file = ""
client-key-file = ""
root-ca-file = ""
```

**Config struct:** `PrivValidatorConfig` in `sei-tendermint/config/config.go`
- `Key` (mapstructure: `key-file`) -- default: `config/priv_validator_key.json`
- `State` (mapstructure: `state-file`) -- default: `data/priv_validator_state.json`
- `ListenAddr` (mapstructure: `laddr`) -- empty = file-based
- `ClientCertificate`, `ClientKey`, `RootCA` -- TLS for gRPC

### 5.3 How the Node Decides: File vs. Remote

In `node/setup.go`, the function `createPrivval` implements the decision:

```
if PrivValidator.ListenAddr != "" {
    protocol, _ = ProtocolAndAddress(ListenAddr)
    switch protocol {
    case "grpc":
        → createAndStartPrivValidatorGRPCClient(...)
    default:
        → createAndStartPrivValidatorSocketClient(...)
    }
} else {
    → use the default FilePV
}
```

And from `makeDefaultPrivval`:
```
if cfg.Mode == ModeValidator {
    → LoadOrGenFilePV(keyFile, stateFile)
} else {
    → return nil (full/seed nodes have no privval)
}
```

The validator is only wired into consensus if `cfg.Mode == ModeValidator`:
```go
if cfg.Mode == config.ModeValidator {
    if privValidator != nil {
        csState.SetPrivValidator(ctx, privValidator)
    }
}
```

### 5.4 Socket-Based Remote Signing (Legacy Protocol)

**Architecture:**
- Node runs a `SignerListenerEndpoint` that listens on TCP or Unix socket
- External KMS (e.g., tmkms) runs a `SignerDialerEndpoint` that dials into the node
- The connection is wrapped in `SecretConnection` (for TCP) providing authenticated
  encryption using X25519 key exchange + ChaCha20-Poly1305

**Protocol:**
- Protobuf `Message` oneOf envelope over length-delimited protobuf frames
- Request/response pairs: PubKeyRequest/PubKeyResponse, SignVoteRequest/SignedVoteResponse,
  SignProposalRequest/SignedProposalResponse, PingRequest/PingResponse
- Each message includes `chain_id` for multi-chain KMS disambiguation

**Connection model:**
1. Node's `SignerListenerEndpoint` calls `listener.Accept()` waiting for incoming connections
2. KMS's `SignerDialerEndpoint` dials the node with retry (default 10 retries, 100ms backoff)
3. For TCP: connection is wrapped in `SecretConnection` using ephemeral X25519 DH + HKDF
4. Ping loop runs every 2/3 of read/write timeout to keep connection alive
5. If ping fails, connection is dropped and listener waits for reconnection

**Timeout defaults:**
- Accept timeout: 3 seconds
- Read/write timeout: 5 seconds
- Ping interval: 3.33 seconds (2/3 of read/write timeout)

**RetrySignerClient:** In production, the socket client is wrapped in
`RetrySignerClient` with 50 retries at 100ms intervals (5s total) for transient
connection failures. Remote signer errors (`RemoteSignerError`) are NOT retried.

**SecretConnection details:**
- STS protocol implementation (Station-to-Station)
- Ephemeral X25519 key exchange for perfect forward secrecy
- HKDF-SHA256 key derivation for send/recv symmetric keys
- ChaCha20-Poly1305 AEAD encryption with incrementing nonces
- Ed25519 signature challenge for mutual authentication
- Frame size: 1024 bytes data + 4 bytes length header + 16 bytes AEAD overhead

### 5.5 gRPC-Based Remote Signing

**Architecture:**
- Node is the gRPC **client**, KMS is the gRPC **server**
- Triggered when `laddr` starts with `grpc://`
- Supports optional mutual TLS via client cert + CA

**Service definition (`service.proto`):**
```protobuf
service PrivValidatorAPI {
    rpc GetPubKey(PubKeyRequest) returns (PubKeyResponse);
    rpc SignVote(SignVoteRequest) returns (SignedVoteResponse);
    rpc SignProposal(SignProposalRequest) returns (SignedProposalResponse);
}
```

**TLS configuration:**
- If `RootCA`, `ClientKey`, and `ClientCertificate` are all set: TLS 1.3 minimum
- Otherwise: insecure connection (with warning log)

**Client configuration:**
- gRPC keepalive: ping every 10s, 2s timeout
- Retry: exponential backoff, max 50 retries, 1s initial timeout
- Max message size: 1MB

**Relevant code:** `sei-tendermint/privval/grpc/`

### 5.6 tmkms Integration

tmkms (Tendermint Key Management System) is the standard external signer for production
Tendermint/Sei validators. It connects to the node's `priv_validator_laddr` socket.

**How it works at the config level:**
1. Set `priv_validator_laddr = "tcp://0.0.0.0:26659"` in `config.toml`
2. tmkms is configured to dial `tcp://validator-host:26659`
3. The connection uses `SecretConnection` with mutual Ed25519 authentication
4. tmkms holds the validator private key (in HSM, YubiHSM, or file)
5. tmkms maintains its own signing state to prevent double-signing

**K8s deployment pattern for tmkms:**
- tmkms runs as a sidecar container or separate deployment
- The validator node exposes port 26659 via a headless Service
- tmkms dials the validator's service hostname
- Key material stored in a Kubernetes Secret mounted into the tmkms container
- For HSMs: the HSM device is exposed via a device plugin

---

## 6. Protobuf Wire Protocol

### 6.1 Message Types (`types.proto`)

```protobuf
message Message {
    oneof sum {
        PubKeyRequest          pub_key_request          = 1;
        PubKeyResponse         pub_key_response         = 2;
        SignVoteRequest        sign_vote_request        = 3;
        SignedVoteResponse     signed_vote_response     = 4;
        SignProposalRequest    sign_proposal_request    = 5;
        SignedProposalResponse signed_proposal_response = 6;
        PingRequest            ping_request             = 7;
        PingResponse           ping_response            = 8;
    }
}
```

### 6.2 Error Codes

```protobuf
enum Errors {
    ERRORS_UNKNOWN             = 0;
    ERRORS_UNEXPECTED_RESPONSE = 1;
    ERRORS_NO_CONNECTION       = 2;
    ERRORS_CONNECTION_TIMEOUT  = 3;
    ERRORS_READ_TIMEOUT        = 4;
    ERRORS_WRITE_TIMEOUT       = 5;
}
```

### 6.3 Request Handler

The `DefaultValidationRequestHandler` in `signer_requestHandler.go`:
1. Validates `chain_id` matches for every request (rejects cross-chain signing)
2. Delegates to the `PrivValidator` implementation for actual signing
3. Returns errors as `RemoteSignerError` in the response (not transport errors)

---

## 7. Operator Key and Keyring

### 7.1 Keyring Backends

The operator key (used for signing transactions, NOT consensus) uses the Cosmos SDK
keyring. Supported backends:

| Backend | Constant | Description |
|---------|----------|-------------|
| `file` | `BackendFile` | Encrypted files in `$HOME/keyring-file/` |
| `os` | `BackendOS` | OS-native keychain (macOS Keychain, Linux libsecret) |
| `test` | `BackendTest` | Unencrypted files in `$HOME/keyring-test/` (NO PASSPHRASE) |
| `memory` | `BackendMemory` | In-memory only (lost on process exit) |
| `kwallet` | `BackendKWallet` | KDE Wallet |
| `pass` | `BackendPass` | pass (the standard unix password manager) |

**Key algorithms supported:**
- `secp256k1` (default for Cosmos accounts, used by Sei)
- `sr25519`
- Ledger hardware wallet (secp256k1, sr25519)

**HD derivation:** BIP-39 mnemonic -> BIP-44 HD path -> private key
- Sei's full BIP-44 path: `m/44'/118'/0'/0/0` (coin type 118 = Cosmos)

### 7.2 Operator Key vs. Validator Key

These are **completely separate keys** with different algorithms:

| Property | Validator Key | Operator Key |
|----------|--------------|--------------|
| Algorithm | Ed25519 (consensus) | secp256k1 (transactions) |
| Purpose | Sign votes + proposals | Sign transactions (create-validator, delegate, etc.) |
| Storage | `priv_validator_key.json` | Keyring (file/os/test backend) |
| Address prefix | hex (20 bytes) | `sei1...` (bech32) |
| Compromise risk | Double-sign slashing + tombstoning | Fund theft |
| Managed by | Tendermint consensus engine | Cosmos SDK CLI |

### 7.3 Bech32 Address Prefixes (Sei-specific)

```go
cfg.SetBech32PrefixForAccount("sei", "seipub")
cfg.SetBech32PrefixForValidator("seivaloper", "seivaloperpub")
cfg.SetBech32PrefixForConsensusNode("seivalcons", "seivalconspub")
```

---

## 8. MsgCreateValidator: Validator Registration

### 8.1 Message Structure

```protobuf
message MsgCreateValidator {
    Description       description         = 1;  // moniker, identity, website, security_contact, details
    CommissionRates   commission           = 2;  // rate, max_rate, max_change_rate
    string            min_self_delegation  = 3;  // minimum self-bond
    string            delegator_address    = 4;  // sei1... (operator account)
    string            validator_address    = 5;  // seivaloper1... (derived from operator)
    google.protobuf.Any pubkey             = 6;  // Ed25519 consensus pubkey (from priv_validator_key.json)
    Coin              value                = 7;  // initial self-delegation amount
}
```

### 8.2 Description Fields

```protobuf
message Description {
    string moniker          = 1;  // human-readable name (REQUIRED)
    string identity         = 2;  // optional keybase.io identity
    string website          = 3;  // optional website URL
    string security_contact = 4;  // optional email
    string details          = 5;  // optional description
}
```

### 8.3 TxCreateValidatorConfig (SDK Builder)

Used by seictl's `generate_gentx.go` to construct the message:

```go
type TxCreateValidatorConfig struct {
    ChainID                 string
    NodeID                  string
    Moniker                 string
    Amount                  string             // e.g., "1000000usei"
    CommissionRate          string             // e.g., "0.1"
    CommissionMaxRate       string             // e.g., "0.2"
    CommissionMaxChangeRate string             // e.g., "0.01"
    MinSelfDelegation       string             // e.g., "1"
    PubKey                  cryptotypes.PubKey // consensus pubkey (Ed25519)
    IP                      string
    P2PPort                 string             // e.g., "26656"
    Website                 string
    SecurityContact         string
    Details                 string
    Identity                string
}
```

### 8.4 How seictl Generates a Gentx

The `generate-gentx` task in seictl follows this sequence:

1. **Create operator key:** `keyring.NewMnemonic("validator", English, BIP44Path, "", hd.Secp256k1)`
   - Backend: `test` (unencrypted, suitable for ephemeral genesis ceremonies)
   - Algorithm: secp256k1
   - Name: `"validator"` (hardcoded constant `validatorKeyName`)

2. **Add genesis account:** Insert auth account + bank balance + EVM address association
   into genesis.json

3. **Load validator identity:** `genutil.InitializeNodeValidatorFiles(cfg)` which returns
   `(nodeID, valPubKey)` by loading the priv_validator_key.json

4. **Build MsgCreateValidator:** Using `stakingcli.BuildCreateValidatorMsg()` with the
   TxCreateValidatorConfig populated from parameters + loaded identity

5. **Sign the transaction:** Using the test keyring with the operator key

6. **Write gentx file:** To `config/gentx/gentx-<nodeID>.json`

---

## 9. Identity Generation in seictl

### 9.1 `generate-identity` Task

**What it does:** Creates the three identity files that `seid init` normally produces.

**Parameters:** `{"chainId": "...", "moniker": "..."}`

**Flow:**
1. `tmcfg.EnsureRoot(homeDir)` -- creates directory structure
2. `genutil.InitializeNodeValidatorFilesFromMnemonic(cfg, "")` -- the empty mnemonic
   means random generation. This single call generates ALL THREE files:
   - `config/node_key.json` (via `config.LoadOrGenNodeKeyID()`)
   - `config/priv_validator_key.json` (via `privval.LoadOrGenFilePV()`)
   - `data/priv_validator_state.json` (via `privval.LoadOrGenFilePV()`)
3. Write `config.toml` with moniker set
4. Write minimal `genesis.json` if none exists

**Idempotency:** Uses marker file `.sei-sidecar-identity-done` to skip if already run.
Also, `LoadOrGenFilePV` checks for existing files before generating new ones.

### 9.2 `InitializeNodeValidatorFilesFromMnemonic` (sei-cosmos/x/genutil/utils.go)

This is the core identity generation function:

```go
func InitializeNodeValidatorFilesFromMnemonic(config *cfg.Config, mnemonic string) (
    nodeID string, valPubKey cryptotypes.PubKey, err error) {

    // 1. Generate/load node key
    nodeKey, err := config.LoadOrGenNodeKeyID()  // -> node_key.json

    // 2. Generate/load validator key
    if len(mnemonic) == 0 {
        filePV, _ = privval.LoadOrGenFilePV(pvKeyFile, pvStateFile)  // random
    } else {
        privKey := tmed25519.GenPrivKeyFromSecret([]byte(mnemonic))  // deterministic
        filePV = privval.NewFilePV(privKey, pvKeyFile, pvStateFile)
    }

    // 3. Convert TM pubkey to SDK pubkey format
    tmValPubKey, _ := filePV.GetPubKey(ctx)
    valPubKey, _ = cryptocodec.FromTmPubKeyInterface(tmValPubKey)

    return nodeID, valPubKey, nil
}
```

**Key insight for the controller:** Deterministic key generation is possible by passing
a mnemonic. `GenPrivKeyFromSecret(secret)` hashes the secret with SHA-256 to get a
32-byte seed, then uses `ed25519.NewKeyFromSeed(seed)`. This could be used to derive
validator keys from a Kubernetes Secret containing a mnemonic/seed.

---

## 10. Slashing Conditions

### 10.1 Sei Default Slashing Parameters

From `sei-cosmos/x/slashing/types/params.go`:

| Parameter | Default Value | Description |
|-----------|--------------|-------------|
| `SignedBlocksWindow` | 108,000 (~12h at 0.4s blocks) | Window for uptime tracking |
| `MinSignedPerWindow` | 0.05 (5%) | Minimum signed blocks to avoid jail |
| `DowntimeJailDuration` | 10 minutes | Jail duration for downtime |
| `SlashFractionDoubleSign` | 0 (0%) | Stake slashed for double-sign |
| `SlashFractionDowntime` | 0 (0%) | Stake slashed for downtime |

**IMPORTANT:** Sei's default configuration has **zero slash fractions** for both
double-sign and downtime. This means validators are jailed but NOT slashed by default.
However, governance can change these parameters at any time.

### 10.2 Double-Sign Prevention

Double signing occurs when a validator signs two different votes (or proposals) at the
same height, round, and step. The `FilePVLastSignState.checkHRS()` function prevents
this at the node level by maintaining a monotonic (height, round, step) watermark.

For remote signers, the KMS (e.g., tmkms) maintains its own state file with the same
HRS watermark. This provides an additional layer of protection.

### 10.3 Downtime/Jailing

A validator is jailed if it misses more than `(1 - MinSignedPerWindow)` of blocks in
the `SignedBlocksWindow`. With Sei defaults: missing more than 95% of 108,000 blocks
triggers jailing. Unjailing requires the operator to send an `MsgUnjail` transaction.

---

## 11. Sei-Specific Modifications

After searching `sei-chain@v0.0.38`, there are **no Sei-specific modifications** to the
validator/signing machinery. Sei uses the standard Tendermint PrivValidator
implementation unchanged. The relevant Sei-specific aspects are:

1. **Bech32 prefixes:** `sei`, `seivaloper`, `seivalcons` (set in `ensureBech32()`)
2. **Slashing defaults:** Zero slash fractions (different from standard Cosmos defaults)
3. **EVM address association:** When creating a genesis validator, seictl also derives
   the Ethereum address from the secp256k1 operator key and adds it to the EVM module
   genesis state
4. **Block time:** ~0.4s (affects slashing window calculations)

---

## 12. Implications for the K8s Controller

### 12.1 Key Storage Requirements

For validator SeiNodes, the controller must handle:

| File | Storage | Persistence | Sensitivity |
|------|---------|-------------|-------------|
| `node_key.json` | PVC (config/) | Persistent | Low (regenerable) |
| `priv_validator_key.json` | PVC (config/) OR K8s Secret | Persistent, immutable | **CRITICAL** |
| `priv_validator_state.json` | PVC (data/) | Persistent, mutable | High (double-sign risk) |
| Operator keyring | K8s Secret or PVC | Persistent | High (fund access) |

### 12.2 Validator Mode Configuration

To run a node in validator mode, the controller must:

1. Set `mode = "validator"` in `config.toml`
2. Ensure `priv_validator_key.json` exists at `config/priv_validator_key.json`
3. Ensure `priv_validator_state.json` exists at `data/priv_validator_state.json`
4. For remote signing: set `laddr` in `[priv-validator]` section

### 12.3 Remote Signing in Kubernetes

**Option A: tmkms as sidecar**
- Deploy tmkms container in the same pod as the validator
- tmkms connects to `tcp://localhost:26659` (node's priv_validator_laddr)
- Key material in a mounted K8s Secret
- State file on a separate PVC or emptyDir

**Option B: tmkms as separate deployment**
- tmkms runs in its own deployment/statefulset
- Connects to the validator's headless service: `tcp://<validator-pod>.<service>:26659`
- More complex networking but better separation of concerns

**Option C: gRPC remote signer**
- Set `laddr = "grpc://<kms-service>:port"` (node is the gRPC client)
- KMS runs as a separate gRPC server
- Supports mutual TLS via `client-certificate-file`, `client-key-file`, `root-ca-file`

### 12.4 Bootstrap Sequence for Validator Nodes

1. **generate-identity** task: Creates node_key.json + priv_validator_key.json + priv_validator_state.json
2. **configure** task: Sets `mode = "validator"` and optionally `priv_validator_laddr`
3. **genesis ceremony** (if applicable): generate-gentx creates operator key + genesis tx
4. **Start seid:** The node reads the key files and enters validator mode

### 12.5 CRD Design Considerations

The SeiNode CRD for validator mode should expose:

```yaml
spec:
  mode: validator
  validator:
    # Remote signing configuration
    remoteSignerAddr: ""  # e.g., "tcp://0.0.0.0:26659" or "grpc://tmkms:port"
    # TLS for gRPC remote signing
    tls:
      clientCertificateSecretRef: {}
      clientKeySecretRef: {}
      rootCASecretRef: {}
    # Key provisioning strategy
    keySource:
      type: generate | secret | remoteSigner
      secretRef: {}  # if type=secret, references K8s Secret with key material
    # Staking configuration (for MsgCreateValidator)
    staking:
      amount: "1000000usei"
      commission:
        rate: "0.1"
        maxRate: "0.2"
        maxChangeRate: "0.01"
      minSelfDelegation: "1"
      description:
        moniker: ""
        identity: ""
        website: ""
        securityContact: ""
        details: ""
```

### 12.6 Safety Invariants

1. **NEVER run two pods with the same priv_validator_key.json simultaneously** -- this
   is the primary double-sign vector. The controller must ensure exactly-once scheduling
   for validator StatefulSets (replicas=1, no parallel pod management).

2. **priv_validator_state.json must be on the same PVC as the node data** -- it tracks
   the last signed HRS. If lost, the node could double-sign on restart.

3. **The validator key file must survive pod rescheduling** -- it's stored on a
   persistent volume. Loss means the validator identity is gone (must re-register with
   new key, losing all delegations).

4. **For remote signing, the `laddr` must be set BEFORE the node starts** -- it's read
   at initialization time in `makeNode()`. Dynamic switching between file and remote
   signing requires a node restart.

5. **The operator keyring (test backend) is only needed during genesis ceremony** -- after
   the chain is live, MsgCreateValidator is sent via the operator's funded account. The
   controller should consider whether to persist this key or treat it as ephemeral.
