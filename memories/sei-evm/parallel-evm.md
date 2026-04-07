---
title: "Sei EVM Integration: Parallel EVM, Pointer Contracts, and EVM RPC"
category: architecture
status: reference
confidence: high
sources:
  sei-chain: "github.com/sei-protocol/sei-chain@v0.0.38"
  sei-cosmos: "github.com/sei-protocol/sei-cosmos@v0.3.66"
  go-ethereum-fork: "github.com/sei-protocol/go-ethereum v1.13.5-sei-17"
files_read:
  - x/evm/module.go
  - x/evm/handler.go
  - x/evm/keeper/keeper.go
  - x/evm/keeper/evm.go
  - x/evm/keeper/msg_server.go
  - x/evm/keeper/pointer.go
  - x/evm/keeper/address.go
  - x/evm/keeper/state.go
  - x/evm/keeper/params.go
  - x/evm/keeper/precompile.go
  - x/evm/state/statedb.go
  - x/evm/state/state.go
  - x/evm/state/balance.go
  - x/evm/state/log.go
  - x/evm/state/utils.go
  - x/evm/types/keys.go
  - x/evm/types/config.go
  - x/evm/types/params.go
  - x/evm/types/message_evm_transaction.go
  - x/evm/types/message_register_pointer.go
  - x/evm/types/ethtx/associate_tx.go
  - x/evm/config/config.go
  - x/evm/ante/preprocess.go
  - x/evm/ante/fee.go
  - x/evm/ante/gas.go
  - x/evm/ante/router.go
  - x/evm/derived/derived.go
  - x/evm/gov.go
  - x/evm/genesis.go
  - x/evm/artifacts/cw20/artifacts.go
  - x/evm/artifacts/cw20/cw20.go (generated binding)
  - x/evm/artifacts/native/native.go (generated binding)
  - x/evm/querier/config.go
  - evmrpc/server.go
  - evmrpc/config.go
  - evmrpc/block.go
  - evmrpc/tx.go
  - evmrpc/state.go
  - evmrpc/send.go
  - precompiles/setup.go
  - precompiles/common/precompiles.go
  - precompiles/bank/bank.go
  - precompiles/pointer/pointer.go
  - sei-cosmos: tasks/scheduler.go
  - sei-cosmos: store/multiversion/store.go
  - sei-cosmos: types/context.go
---

# Sei EVM Integration

## 1. Architecture Overview

Sei runs an Ethereum-compatible EVM as a **native Cosmos SDK module** (`x/evm`), not as an external sidechain or virtual machine bridge. The EVM module is registered alongside all other Cosmos modules (staking, bank, wasm, etc.) and participates in the standard ABCI lifecycle (BeginBlock, DeliverTx, EndBlock). This is fundamentally different from Ethermint/Evmos, which replace the Cosmos transaction pipeline with an Ethereum-first model.

**Key architectural properties:**
- EVM transactions are wrapped inside Cosmos SDK transactions as `MsgEVMTransaction`
- EVM state (contract storage, code, nonces) lives in the Cosmos SDK IAVL store under the `evm` module's store key
- EVM balances ARE Cosmos bank module balances -- there is no separate EVM token
- Address association links Ethereum addresses (0x...) to Cosmos bech32 addresses (sei1...)
- Precompiled contracts bridge EVM calls to Cosmos module functionality

### Module registration

The EVM module (`x/evm`) registers as a standard `module.AppModule` in `app/app.go`. It has its own:
- Store key: `"evm"`
- Params subspace with `PriorityNormalizer`, `BaseFeePerGas`, `MinimumFeePerGas`
- Message types: `MsgEVMTransaction`, `MsgSend`, `MsgRegisterPointer`
- GRPC query service
- Genesis import/export
- Governance proposal types for pointer registration


## 2. EVM Transaction Processing

### Transaction lifecycle

1. **Encoding**: An Ethereum RLP-encoded transaction arrives via the EVM JSON-RPC server (`evmrpc/send.go`). It is deserialized into `ethtypes.Transaction`, wrapped in `ethtx.TxData`, then wrapped in `MsgEVMTransaction`, and finally wrapped in a Cosmos SDK transaction envelope using `txConfig.TxBuilder`.

2. **Ante handler routing**: The `EVMRouterDecorator` (`ante/router.go`) checks if the transaction contains a `MsgEVMTransaction`. If yes, it routes to the EVM-specific ante handler chain instead of the default Cosmos ante handler.

3. **EVM ante handler chain** (in order):
   - `EVMPreprocessDecorator`: Recovers the sender's public key from the ECDSA signature (V, R, S), derives both the EVM address (Keccak-256 of uncompressed pubkey) and the Sei bech32 address (from the compressed secp256k1 pubkey). Sets `msg.Derived` with sender addresses and pubkey. Automatically creates address association if not already associated. Sets gas meter to **infinite** for ante processing.
   - `EVMFeeCheckDecorator`: Validates that `GasFeeCap >= BaseFeePerGas` and `GasFeeCap >= MinimumFeePerGas`. Calls `st.BuyGas()` to debit the fee upfront from the sender's account via the stateDB. Calculates transaction priority.
   - `GasLimitDecorator`: Converts the EVM gas limit to Sei gas units using `PriorityNormalizer` and sets the gas meter accordingly (this gas meter value is returned to Tendermint mempool during CheckTx).

4. **Message server execution** (`keeper/msg_server.go:EVMTransaction`):
   - Resets gas meter to **infinite** again (EVM manages its own gas internally)
   - Creates a new `state.DBImpl` as the EVM state database
   - Calls `applyEVMMessage()` which:
     - Gets the VM block context (coinbase, block number, timestamp, etc.)
     - Creates `vm.EVM` instance with Sei's chain config
     - Creates `core.StateTransition` with `feeAlreadyCharged=true`
     - Calls `st.TransitionDb()` -- standard go-ethereum state transition
   - After execution, writes receipt, finalizes stateDB (flushing cached writes)
   - Converts EVM gas used back to Sei gas units and charges the original gas meter

5. **EndBlock**: Processes deferred info from all EVM transactions in the block:
   - Collects coinbase fees from per-transaction temporary addresses
   - Sends collected fees to the block proposer (coinbase)
   - Handles surplus (gas fee accounting differences)
   - Stores bloom filter and transaction hashes per block height

### Associate transactions

A special transaction type `AssociateTx` allows linking an Ethereum address to a Sei address without executing any EVM logic. The user signs a custom message with their Ethereum private key. The ante handler verifies the signature, derives both addresses, and calls `SetAddressMapping()`. The message server returns immediately (no-op). This is used for accounts that want to interact with both Cosmos and EVM sides.


## 3. Pointer Contracts -- The Bidirectional Bridge

Pointer contracts are Sei's core innovation for interoperability between EVM and Cosmos worlds. They create **bidirectional proxy contracts** that allow assets from one side to appear natively on the other.

### Types of pointer contracts

There are **five** pointer directions stored under `PointerRegistryPrefix` (0x15):

| Direction | Key Prefix | Description |
|-----------|------------|-------------|
| ERC20 -> Native | `0x0` | EVM ERC-20 contract that represents a Cosmos native token (e.g., usei, ibc/...) |
| ERC20 -> CW20 | `0x1` | EVM ERC-20 contract that represents a CosmWasm CW20 token |
| ERC721 -> CW721 | `0x2` | EVM ERC-721 contract that represents a CosmWasm CW721 NFT |
| CW20 -> ERC20 | `0x3` | CosmWasm CW20 contract that represents an EVM ERC-20 token |
| CW721 -> ERC721 | `0x4` | CosmWasm CW721 contract that represents an EVM ERC-721 NFT |

### How ERC20 Native pointers work

When a governance proposal (`AddERCNativePointerProposal`) or the pointer precompile registers a pointer for a native Cosmos denom:

1. An ERC-20 Solidity contract (`NativeSeiTokensERC20`) is deployed to the EVM. This contract's ABI includes standard ERC-20 methods (transfer, approve, balanceOf, totalSupply, etc.) plus a `denom()` getter and a reference to the `BankPrecompile`.

2. The contract's `balanceOf()` calls the **bank precompile** (0x1001) to query the Cosmos bank module balance for the given denom and address.

3. The contract's `transfer()` calls the bank precompile's `send()` method, which internally calls `bankKeeper.SendCoins()`.

4. The mapping is stored bidirectionally:
   - Forward: `PointerERC20NativeKey(denom)` -> EVM contract address
   - Reverse: `PointerReverseRegistryKey(evmAddr)` -> denom string

5. **Anti-pointer-to-pointer guard**: `cwAddressIsPointer()` and `evmAddressIsPointer()` check the reverse registry to prevent creating a pointer to something that is itself already a pointer.

### How ERC20-CW20 pointers work

For CW20 tokens visible in the EVM:

1. An ERC-20 Solidity contract (`CW20ERC20Pointer`) is deployed. Its constructor takes the CW20 contract address.

2. The contract uses the **wasmd precompile** (0x1002) to execute CW20 queries and messages. For example, `balanceOf()` calls `wasmd.query()` with a CW20 `{balance: {address: "..."}}` query.

3. The contract also uses the **addr precompile** (0x1004) to translate between EVM addresses and Sei addresses, and the **json precompile** (0x1003) to serialize/deserialize JSON for wasmd calls.

### How CW20-ERC20 pointers work (reverse direction)

For ERC-20 tokens visible in CosmWasm:

1. A CosmWasm contract (from `artifacts/erc20/cwerc20.wasm`) is instantiated. It takes the ERC-20 address as an init parameter.

2. This CW20-compatible contract uses `MsgInternalEVMCall` to call back into the EVM to query/transfer the underlying ERC-20 token.

3. Stored with `PointerCW20ERC20Key(erc20Address)` -> CW20 contract address.

### Pointer creation mechanisms

1. **Governance proposals**: `AddERCNativePointerProposal`, `AddERCCW20PointerProposal`, `AddERCCW721PointerProposal` -- handled in `gov.go`

2. **Pointer precompile** (0x100b): EVM contracts can call `addNativePointer()`, `addCW20Pointer()`, `addCW721Pointer()` from Solidity

3. **MsgRegisterPointer** transaction: For CW20->ERC20 and CW721->ERC721 pointers. Instantiates the CW pointer contract and registers the mapping.

### Versioning

Every pointer has a version number. The keeper stores pointers keyed by `[prefix][identifier][version_uint16_be]`. `GetPointerInfo()` does a **reverse iterator** to find the highest version. Newer versions override older ones. This allows upgrading pointer contract implementations.


## 4. Parallel Execution via OCC (Optimistic Concurrency Control)

### OCC is at the Cosmos SDK level, not EVM-specific

Sei's parallel execution is implemented in `sei-cosmos` (the forked Cosmos SDK), not in the EVM module itself. The OCC scheduler lives in `tasks/scheduler.go` and operates at the `DeliverTx` level -- it parallelizes ALL transaction types (Cosmos native, CosmWasm, and EVM).

### How the OCC scheduler works

1. **Multi-version store**: Each KV store key gets a `MultiVersionStore` (`store/multiversion/store.go`) that tracks reads and writes per transaction index and incarnation. This is conceptually similar to software transactional memory.

2. **Execution phase**: All transactions in a block are dispatched to a worker pool concurrently. Each transaction gets a `VersionIndexedStore` that records its read set and write set.

3. **Validation phase**: After execution, each transaction is validated by checking if any of its reads have been invalidated by a lower-indexed transaction's writes. `ValidateTransactionState()` compares the read set against the multi-version store.

4. **Conflict resolution**: If validation fails, the transaction is re-executed (with an incremented incarnation number). Dependencies are tracked so re-execution waits for conflicting lower-indexed transactions to complete.

5. **Fallback to sequential**: If the maximum incarnation exceeds `maximumIterations` (10), the scheduler falls back to sequential execution for the remaining transactions.

6. **Final commit**: Once all transactions are validated, `WriteLatestToStore()` flushes the multi-version store to the underlying IAVL store.

### How EVM participates in OCC

EVM transactions participate in OCC like any other Cosmos transaction. The key integration points:

- **Access control annotations**: The `EVMPreprocessDecorator.AnteDeps()` method declares the KV store access patterns for each EVM transaction (read/write to address mappings, bank balances, nonces). These access operations use Cosmos SDK's `ResourceType_KV_EVM_S2E`, `ResourceType_KV_EVM_E2S`, `ResourceType_KV_EVM_NONCE`, etc.

- **StateDB writes through Cosmos store**: The `state.DBImpl` writes all EVM state changes (storage, balances, nonces, code) through the Cosmos SDK `KVStore` interface. In OCC mode, this KVStore is backed by a `VersionIndexedStore` that tracks conflicts.

- **Deferred info via sync.Map**: The keeper uses `sync.Map` for `deferredInfo` to safely accumulate per-transaction bloom filters, tx hashes, and surplus amounts from concurrent execution.

- **Per-tx coinbase addresses**: Each EVM transaction gets a unique temporary coinbase address (`evm_coinbase` + tx_index) for fee collection, avoiding contention on a single fee collector account.

### EVM-specific parallel concern: EVM->CW->EVM not supported

The code explicitly rejects re-entrant cross-VM calls: `CallEVM()` checks `ctx.IsEVM()` and returns an error if true. The context's `evm` flag (from `sei-cosmos`) prevents EVM->CosmWasm->EVM call chains, which could create complex concurrency issues.


## 5. EVM State and Cosmos State Coexistence

### Shared store, separate key prefixes

EVM state lives in the **same IAVL tree** as all other Cosmos module state, under the `"evm"` store key. Key prefixes partition the data:

| Prefix | Content |
|--------|---------|
| `0x01` | EVM address -> Sei address mapping |
| `0x02` | Sei address -> EVM address mapping |
| `0x03` | Contract storage (per-address key-value pairs) |
| `0x07` | Contract code (bytecode) |
| `0x08` | Code hashes |
| `0x09` | Code sizes |
| `0x0a` | Nonces |
| `0x0b` | Transaction receipts |
| `0x0d` | Block bloom filters |
| `0x0e` | Transaction hashes per block height |
| `0x15` | Pointer registry |
| `0x17` | Pointer reverse registry |

### Balance is the Cosmos bank module

EVM balances are NOT stored in the EVM module's store. Instead, `state.DBImpl.GetBalance()` calls `bankKeeper.GetBalance(ctx, seiAddr, "usei")` and `bankKeeper.GetWeiBalance(ctx, seiAddr)` and combines them:

```
balance = usei * 10^12 + wei
```

This means:
- 1 usei = 10^12 wei (called "swei" internally)
- EVM sees 18-decimal ETH-like balances
- Cosmos sees 6-decimal usei balances
- The `BankKeeper.GetWeiBalance()` and `AddWei()`/`SubWei()` methods are Sei-specific additions to the bank module in `sei-cosmos`

### StateDB implementation

`state.DBImpl` implements go-ethereum's `vm.StateDB` interface by delegating to the Cosmos KV store:
- `GetState(addr, key)` -> `keeper.PrefixStore(ctx, types.StateKey(addr)).Get(key[:])`
- `SetState(addr, key, val)` -> `keeper.PrefixStore(ctx, types.StateKey(addr)).Set(key[:], val[:])`
- `GetBalance(addr)` -> bank module query (as described above)
- `AddBalance/SubBalance` -> `bankKeeper.AddCoins()`/`bankKeeper.SubUnlockedCoins()` + wei operations
- Snapshots -> `ctx.MultiStore().CacheMultiStore()` (Cosmos cache multistore layering)
- Finalize -> flushes cache multistores in reverse order

### Surplus accounting

During EVM execution, balance changes through the bank module may not perfectly match EVM expectations (due to the usei/wei split). A "surplus" counter in `TemporaryState` tracks the difference. `SubBalance` adds to surplus (tokens leaving EVM accounting), `AddBalance` subtracts from surplus. The net surplus is collected in EndBlock and sent to the EVM module account.


## 6. Gas Metering Across EVM and Cosmos

### PriorityNormalizer -- the conversion factor

The `PriorityNormalizer` parameter (default: `1`) converts between EVM gas units and Sei gas units:

```
sei_gas = evm_gas * PriorityNormalizer
```

This coefficient is used in multiple places:

1. **Ante handler** (`GasLimitDecorator`): Converts EVM gas limit to Sei gas limit for the Cosmos gas meter
2. **Message server**: After EVM execution, converts `res.UsedGas` back to Sei units and charges the original gas meter
3. **Priority calculation**: `effective_gas_price / PriorityNormalizer` determines transaction priority in Tendermint mempool
4. **CW->EVM calls**: `getEvmGasLimitFromCtx()` converts remaining Sei gas to EVM gas limit

### Gas flow for EVM transactions

1. Ante: Gas meter set to infinite (for ante processing)
2. Ante fee check: `st.BuyGas()` debits full gas * gasPrice from sender
3. Ante gas limit: Gas meter set to `evmGasLimit * PriorityNormalizer`
4. Message server: Gas meter reset to infinite (EVM manages gas internally)
5. EVM execution: go-ethereum's `StateTransition.TransitionDb()` manages gas
6. Post-execution: `adjustedGasUsed = PriorityNormalizer * evmGasUsed` charged to original gas meter
7. Unused gas is refunded during state finalization

### Gas for precompile calls

Precompiles use Cosmos KV gas model: `ReadCostFlat + ReadCostPerByte * len(args)` for reads, `WriteCostFlat + WriteCostPerByte * len(args)` for writes. The `GetRemainingGas()` helper converts remaining Sei gas to EVM gas for precompile execution budgets.


## 7. EVM RPC Layer

### Server architecture

Sei runs a **separate HTTP/WebSocket server** for Ethereum JSON-RPC, independent of the Tendermint RPC and Cosmos gRPC endpoints.

- HTTP default port: **8545**
- WebSocket default port: **8546**
- Built on go-ethereum's `rpc` package

### API namespaces and services

| Namespace | Service | Key methods |
|-----------|---------|-------------|
| `eth` | `BlockAPI` | `eth_getBlockByHash`, `eth_getBlockByNumber`, `eth_getBlockTransactionCountBy*` |
| `eth` | `TransactionAPI` | `eth_getTransactionReceipt`, `eth_getTransactionByHash`, `eth_getTransactionByBlock*` |
| `eth` | `StateAPI` | `eth_getBalance`, `eth_getCode`, `eth_getStorageAt`, `eth_getProof` |
| `eth` | `InfoAPI` | `eth_chainId`, `eth_blockNumber`, `eth_gasPrice`, `eth_feeHistory` |
| `eth` | `SendAPI` | `eth_sendRawTransaction`, `eth_signTransaction` |
| `eth` | `SimulationAPI` | `eth_call`, `eth_estimateGas` |
| `eth` | `FilterAPI` | `eth_getLogs`, `eth_newFilter`, `eth_getFilterChanges` |
| `eth` | `SubscriptionAPI` (WS only) | `eth_subscribe` (newHeads, logs, newPendingTransactions) |
| `net` | `NetAPI` | `net_version`, `net_listening`, `net_peerCount` |
| `txpool` | `TxPoolAPI` | `txpool_content`, `txpool_status` |
| `web3` | `Web3API` | `web3_clientVersion`, `web3_sha3` |
| `debug` | `DebugAPI` | `debug_traceTransaction`, `debug_traceBlockByNumber` |
| `sei` | `AssociationAPI` | Sei-specific address association methods |
| `test` | `TestAPI` | Only on non-live chain IDs; for integration testing overrides |

### Transaction flow through RPC

`eth_sendRawTransaction`:
1. Deserialize RLP bytes into `ethtypes.Transaction`
2. Convert to `ethtx.TxData` (Sei's protobuf wrapper)
3. Wrap in `MsgEVMTransaction`
4. Encode as Cosmos SDK transaction
5. Broadcast via Tendermint RPC (`BroadcastTx` or `BroadcastTxCommit` if slow mode)

Receipts are stored in the EVM module's store (prefix `0x0b`) and queried by the `TransactionAPI`.

### State queries

`eth_getBalance`, `eth_getCode`, `eth_getStorageAt` create a `state.DBImpl` with the requested block height's context and query directly from the Cosmos store. This allows historical state queries as long as the IAVL pruning settings retain the data.

### Configuration

```toml
[evm]
http_enabled = true
http_port = 8545
ws_enabled = true
ws_port = 8546
simulation_gas_limit = 10000000
simulation_evm_timeout = "60s"
filter_timeout = "120s"
max_log_no_block = 10000
max_blocks_for_log = 2000
max_subscriptions_new_head = 10000
slow = false  # if true, uses BroadcastTxCommit (synchronous)
```


## 8. EVM-Specific Precompiled Contracts

Sei registers **11 custom precompiled contracts** at fixed addresses in the EVM, enabling Solidity contracts to call Cosmos module functionality directly:

| Address | Name | Functionality |
|---------|------|---------------|
| `0x...1001` | **bank** | Send tokens, query balances/supply/name/symbol/decimals for any Cosmos denom |
| `0x...1002` | **wasmd** | Execute and query CosmWasm contracts from Solidity |
| `0x...1003` | **json** | JSON serialize/deserialize (needed for wasmd message formatting) |
| `0x...1004` | **addr** | Translate between EVM (0x) and Sei (sei1) addresses |
| `0x...1005` | **staking** | Delegate, undelegate, redelegate, query validators |
| `0x...1006` | **gov** | Submit and vote on governance proposals |
| `0x...1007` | **distribution** | Claim staking rewards, query delegation rewards |
| `0x...1008` | **oracle** | Query oracle exchange rates |
| `0x...1009` | **ibc** | IBC transfers from Solidity |
| `0x...100A` | **pointerview** | Query pointer registry (check if a pointer exists) |
| `0x...100B` | **pointer** | Register new pointer contracts (addNativePointer, addCW20Pointer, addCW721Pointer) |

These are registered globally in go-ethereum's precompile maps (`vm.PrecompiledContractsCancun`, etc.) during `app.go` initialization via `precompiles.InitializePrecompiles()`.

Payable precompiles (bank, staking, gov, wasmd) receive value transfers and handle refund/debit accounting through `HandlePaymentUsei()` and `HandlePaymentUseiWei()`.


## 9. EVM Chain Configuration

### Chain IDs

| Cosmos Chain ID | EVM Chain ID | Network |
|-----------------|-------------|---------|
| `pacific-1` | 1329 (0x531) | Mainnet |
| `atlantic-2` | 1328 (0x530) | Testnet |
| `arctic-1` | 713715 | Devnet |
| Other | 713715 (default) | Local/custom |

### Ethereum upgrade timeline

All Ethereum upgrades from Homestead through Shanghai are activated at genesis (block 0). Cancun activation is configurable via `ChainConfig.CancunTime` (default: 0, meaning genesis). Prague and Verkle are disabled by default (set to -1).

### EVM module parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `PriorityNormalizer` | 1 | Sei gas / EVM gas conversion ratio |
| `BaseFeePerGas` | 0 | Fee per gas that is burned (not paid to validator) |
| `MinimumFeePerGas` | 1000000000 (1 Gwei) | Minimum allowed gas price |
| `WhitelistedCwCodeHashesForDelegateCall` | `[]` | Code hashes allowed to make EVM delegate calls |

### EVM query config

| Parameter | Default | Description |
|-----------|---------|-------------|
| `evm_query_gas_limit` | 300000 | Gas limit for read-only EVM queries |


## 10. Sei-Specific EVM Differentiators vs Ethermint/Evmos

### 1. Native integration, not a replacement

Ethermint replaces the Cosmos transaction pipeline; Sei adds EVM alongside it. Both Cosmos-native and EVM transactions coexist in the same block, processed by the same consensus.

### 2. Shared bank module balances

Sei uses the Cosmos bank module for ALL balances. There is no separate EVM balance store. The `sei-cosmos` fork adds `GetWeiBalance()`/`AddWei()`/`SubWei()` to the bank keeper for sub-usei precision (12 extra decimal places). This means a single unified token (SEI) works across both execution environments.

### 3. Pointer contracts for bidirectional token interop

No other Cosmos EVM chain has the pointer mechanism. Pointers create real contract instances (Solidity or CW) that proxy calls to the other VM, making any token appear native in both environments simultaneously.

### 4. OCC parallel execution

Transaction parallelism via optimistic concurrency control at the Cosmos SDK level. EVM transactions participate alongside Cosmos-native transactions. The multi-version store detects conflicts and re-executes only conflicting transactions.

### 5. Custom precompiles for full Cosmos access

11 precompiled contracts give Solidity direct access to: bank, wasmd, staking, governance, distribution, oracle, IBC, address translation, JSON handling, and pointer management. This is far more comprehensive than Ethermint's precompile set.

### 6. Address association model

Sei's dual-address model (Sei bech32 + EVM hex) with automatic association on first EVM transaction. Balance migration happens automatically when addresses are associated -- funds at the "cast" address (EVM bytes interpreted as Cosmos address) are moved to the true associated Cosmos address.

### 7. No re-entrant cross-VM calls

Sei explicitly blocks EVM->CW->EVM call patterns (`ctx.IsEVM()` check), which simplifies the concurrency model and avoids complex state management issues that would arise from cross-VM re-entrancy.

### 8. Per-transaction coinbase addresses

Instead of a single fee collector that becomes a concurrency bottleneck, each EVM transaction deposits fees into a unique temporary address (`evm_coinbase` + tx_index). EndBlock aggregates all fees. This is critical for parallel execution.

### 9. Forked go-ethereum (v1.13.5-sei-17)

Sei uses a **forked go-ethereum** (`github.com/sei-protocol/go-ethereum v1.13.5-sei-17`) based on Geth 1.13.5 (Cancun-ready). The fork integrates with Sei's precompile registration, state management, and the `feeAlreadyCharged` flag in `StateTransition`.
