---
title: Validator Lifecycle on Sei
category: consensus
sources:
  sei-cosmos: v0.3.66
  sei-tendermint: v0.6.4
files_read:
  - x/staking/types/msg.go
  - x/staking/types/params.go
  - x/staking/types/commission.go
  - x/staking/types/errors.go
  - x/staking/types/staking.pb.go (Validator struct, BondStatus enum, Description struct)
  - x/staking/keeper/msg_server.go
  - x/staking/keeper/val_state_change.go
  - x/staking/keeper/validator.go
  - x/staking/keeper/delegation.go
  - x/staking/keeper/slash.go
  - x/staking/keeper/params.go
  - x/staking/abci.go
  - x/slashing/abci.go
  - x/slashing/types/params.go
  - x/slashing/types/msg.go
  - x/slashing/keeper/infractions.go
  - x/slashing/keeper/unjail.go
  - x/evidence/abci.go
  - x/evidence/keeper/infraction.go
  - x/evidence/types/params.go
  - x/gov/types/msgs.go
  - x/gov/types/params.go
  - x/gov/keeper/vote.go
  - x/gov/keeper/tally.go
  - x/gov/keeper/msg_server.go
  - x/distribution/abci.go
  - x/distribution/keeper/allocation.go
  - x/distribution/types/params.go (defaults only)
  - sei-tendermint: internal/evidence/verify.go
updated: 2026-04-07
---

# Validator Lifecycle on Sei

Complete reference for on-chain validator operations from creation through active operation, covering all transactions, state transitions, and Sei-specific parameters.

---

## 1. Validator Creation (MsgCreateValidator)

### Required Fields

| Field | Type | Constraints |
|---|---|---|
| `ValidatorAddress` | `sdk.ValAddress` | Must equal `DelegatorAddress` (self-delegation) |
| `DelegatorAddress` | `sdk.AccAddress` | Same identity as the operator |
| `Pubkey` | `cryptotypes.PubKey` | Consensus key; must not already be registered. Pubkey type must be in `ConsensusParams.Validator.PubKeyTypes` |
| `Value` | `sdk.Coin` | Initial self-delegation amount. Denom must match `BondDenom` (typically `usei`). Must be >= `MinSelfDelegation` |
| `Description` | `Description` | Must be non-empty. Contains `Moniker`, `Identity`, `Website`, `SecurityContact`, `Details` |
| `Commission` | `CommissionRates` | `Rate`, `MaxRate`, `MaxChangeRate`. Rate must be >= chain `MinCommissionRate` |
| `MinSelfDelegation` | `sdk.Int` | Must be positive |

### What Happens On-Chain

1. **Uniqueness checks**: operator address and consensus pubkey must both be unregistered.
2. **Commission enforcement**: `msg.Commission.Rate >= MinCommissionRate(ctx)` (Sei default: 5%).
3. **Validator record created** with status `Unbonded`, stored in three indices:
   - Main validator store (by operator address)
   - By consensus address (for slashing lookups)
   - By power index (for active set selection)
4. **Hook fired**: `AfterValidatorCreated`.
5. **Self-delegation executed**: `Delegate(ctx, delegatorAddr, bondAmt, Unbonded, validator, true)` -- coins move from the operator's wallet to the staking module's not-bonded pool.
6. Validator does NOT immediately enter the active set. It remains `Unbonded` until the next `EndBlock` runs `ApplyAndReturnValidatorSetUpdates`.

### Signing Key

The transaction is signed by the **operator key** (the `DelegatorAddress` / `ValidatorAddress` account key). The `Pubkey` field is the **consensus key** used by Tendermint for block signing -- it is a separate key.

---

## 2. Entering the Active Set

### How Selection Works

At every `EndBlock`, the staking module calls `BlockValidatorUpdates` -> `ApplyAndReturnValidatorSetUpdates`:

1. Iterate all validators by **descending power** (the power-index store).
2. Take the top `MaxValidators` validators that are:
   - **Not jailed** (jailed validators are excluded from the power index entirely)
   - **Non-zero consensus power** (tokens / PowerReduction > 0)
3. For each selected validator:
   - If currently `Unbonded` -> transition to `Bonded` (tokens move from NotBonded pool to Bonded pool)
   - If currently `Unbonding` -> transition to `Bonded`
   - If already `Bonded` -> no state change
4. Validators that were in the previous active set but are NOT in the new top-N become `Unbonding` (begin unbonding timer).
5. Return `[]abci.ValidatorUpdate` to Tendermint with each validator's consensus power.

### Selection is Purely Stake-Weighted

There is no minimum stake to become active -- if a validator has non-zero power and is in the top `MaxValidators`, it enters the set.

### Sei-Specific: MaxVotingPowerRatio

Sei adds a **max voting power ratio** check. During delegation (`Delegate()`), if total bonded power is >= `MaxVotingPowerEnforcementThreshold`, a single validator's power cannot exceed `MaxVotingPowerRatio` of total power. If exceeded, the delegation is rejected with `ErrExceedMaxVotingPowerRatio`.

This is enforced at delegation time, not at EndBlock.

### Default Staking Parameters

| Parameter | Default | Description |
|---|---|---|
| `MaxValidators` | **35** | Maximum active set size (Sei-specific, standard Cosmos is 100) |
| `UnbondingTime` | **21 days** (3 weeks) | `time.Hour * 24 * 7 * 3` |
| `MaxEntries` | **7** | Max simultaneous unbonding/redelegation entries per pair |
| `BondDenom` | `sdk.DefaultBondDenom` | Typically `usei` |
| `MinCommissionRate` | **5%** (`0.05`) | Sei-specific, standard Cosmos is 0% |
| `MaxVotingPowerRatio` | `sdk.DefaultMaxVotingPowerRatio` | Sei-specific cap on single validator power |
| `MaxVotingPowerEnforcementThreshold` | `sdk.DefaultMaxVotingPowerEnforcementThreshold` | Total power must exceed this before ratio is enforced |
| `HistoricalEntries` | **10000** | Number of historical validator sets kept |
| `PowerReduction` | `sdk.DefaultPowerReduction` (10^6) | Tokens per unit of consensus power |

---

## 3. Delegation

### MsgDelegate

Any account can delegate to any registered validator:

```
MsgDelegate {
    DelegatorAddress string
    ValidatorAddress string
    Amount           sdk.Coin   // must match BondDenom
}
```

Signed by: the **delegator's account key**.

### Mechanics

1. Validator must exist.
2. Coin denom must match `BondDenom`.
3. **MaxVotingPowerRatio** check is performed (see above).
4. Exchange rate: delegator receives **shares** proportional to `bondAmt / validator.Tokens * validator.DelegatorShares`. This means the share price changes after slashing.
5. If the validator is `Bonded`, tokens go to the Bonded pool. If `Unbonding`/`Unbonded`, tokens go to the NotBonded pool.
6. The validator's total tokens and total shares increase.

### Effect on Validator Power

Yes, delegations directly affect validator power. Consensus power = `validator.Tokens / PowerReduction`. More delegation = more tokens = more power = higher rank in the active set.

### Redelegation (MsgBeginRedelegate)

Move stake between validators without unbonding:

- **No waiting period for the delegator** -- stake is immediately active on the destination validator.
- However, **transitive redelegation is blocked**: you cannot redelegate tokens that were already redelegated until the first redelegation completes (after `UnbondingTime`).
- Max `MaxEntries` (7) simultaneous redelegation entries per (delegator, src, dst) triplet.

---

## 4. Jailing

### Cause 1: Downtime (Liveness Fault)

Detected by the **slashing module's `BeginBlocker`**. For every active validator:

1. Track signed/missed blocks in a circular bit array of size `SignedBlocksWindow`.
2. After the validator has been active for at least `SignedBlocksWindow` blocks:
   - If `MissedBlocksCounter > SignedBlocksWindow - MinSignedPerWindow` (i.e., missed more than the allowed maximum), the validator is slashed and jailed.

**Action on downtime jail:**
- Slash by `SlashFractionDowntime` (Sei default: **0%** -- no token slash by default)
- Set `Jailed = true`
- Remove from power index (excluded from active set at next EndBlock)
- Set `JailedUntil = blockTime + DowntimeJailDuration`
- Reset `MissedBlocksCounter` and `IndexOffset` to 0

### Cause 2: Double Signing (Equivocation)

Detected by **Tendermint** and reported via `RequestBeginBlock.ByzantineValidators`. The `x/evidence` module's `BeginBlocker` processes it:

1. Tendermint detects `DuplicateVoteEvidence` or `LightClientAttack` and includes it in the next block's `BeginBlock` request.
2. The evidence module calls `HandleEquivocationEvidence`:
   - Evidence must not be stale (checked against `MaxAgeDuration` and `MaxAgeNumBlocks` from consensus params)
   - Validator must not already be tombstoned
   - Slash by `SlashFractionDoubleSign` (Sei default: **0%** -- no token slash by default)
   - Jail the validator
   - **Tombstone** the validator: set `JailedUntil = DoubleSignJailEndTime` (Dec 31, 9999 23:59:59 UTC)
   - Tombstoned validators can **never** be unjailed

### Cause 3: Falling Below MinSelfDelegation

When undelegation causes a validator's self-delegation to drop below `MinSelfDelegation`, the validator is jailed. This validator can unjail once their self-delegation is restored above the minimum.

### Default Slashing Parameters

| Parameter | Default | Description |
|---|---|---|
| `SignedBlocksWindow` | **108,000** | ~12 hours at ~0.4s block times |
| `MinSignedPerWindow` | **5%** (`0.05`) | Must sign at least 5% of blocks in the window. Meaning: can miss up to 95% (102,600 blocks) before jailing |
| `DowntimeJailDuration` | **10 minutes** | `60 * 10 * time.Second` |
| `SlashFractionDoubleSign` | **0%** (`0.0`) | Sei default: no token burn for double-sign (governance can change) |
| `SlashFractionDowntime` | **0%** (`0.0`) | Sei default: no token burn for downtime (governance can change) |

---

## 5. Unjailing (MsgUnjail)

### Message

```
MsgUnjail {
    ValidatorAddr string   // validator operator address (bech32)
}
```

Signed by: the **validator operator key** (the `ValAddress` converted to `AccAddress`).

### Preconditions

1. Validator must exist.
2. Validator must be **jailed** (`IsJailed() == true`).
3. Self-delegation must exist and be >= `MinSelfDelegation`.
4. If signing info exists (validator was previously bonded):
   - Must NOT be **tombstoned** (tombstoned = permanent jail from double-signing).
   - Current block time must be >= `JailedUntil` (jail period must have elapsed).
5. If no signing info exists (validator was jailed before ever being bonded, e.g., fell below MinSelfDelegation), the validator can unjail immediately once conditions are met.

### What Happens

1. `Jailed` flag set to `false`.
2. Validator re-inserted into the power index.
3. At the next `EndBlock`, if the validator's power is in the top `MaxValidators`, it re-enters the active set.

---

## 6. Validator Status State Machine

```
                    MsgCreateValidator
                          |
                          v
                     [Unbonded]
                      |       ^
       (top N by     |        | (all delegations removed,
        power at     |        |  unbonding period completes)
        EndBlock)    |        |
                     v        |
                    [Bonded] ---> [Unbonding]
                      ^              |
                      |              | (re-enters top N
                      |              |  before unbonding
                      +--------------+   completes)
```

### Status Transitions

| From | To | Trigger |
|---|---|---|
| `Unbonded` | `Bonded` | EndBlock: validator enters top MaxValidators by power |
| `Unbonding` | `Bonded` | EndBlock: validator re-enters top MaxValidators |
| `Bonded` | `Unbonding` | EndBlock: validator falls out of top MaxValidators (or jailed) |
| `Unbonding` | `Unbonded` | Unbonding period completes (`UnbondAllMatureValidators`) |

Validators can only be **removed from the store** when they are `Unbonded` AND have zero tokens.

---

## 7. Unbonding

### Validator Unbonding

When a bonded validator leaves the active set:
1. Status set to `Unbonding`.
2. `UnbondingHeight` = current block height.
3. `UnbondingTime` = current block time + `UnbondingTime` parameter (21 days).
4. Inserted into the unbonding validator queue.
5. Tokens move from the Bonded pool to the NotBonded pool.
6. At maturity, status changes to `Unbonded`.

### Delegation Unbonding (MsgUndelegate)

```
MsgUndelegate {
    DelegatorAddress string
    ValidatorAddress string
    Amount           sdk.Coin
}
```

1. Shares are converted to tokens and removed from the validator.
2. An `UnbondingDelegation` entry is created with completion time = block time + `UnbondingTime`.
3. Tokens are held in the NotBonded pool during the unbonding period.
4. At maturity (processed by `DequeueAllMatureUBDQueue` in `EndBlock`), tokens are returned to the delegator's account.
5. During unbonding, tokens can still be slashed if the infraction occurred before unbonding began.

---

## 8. Commission

### Structure

```go
CommissionRates {
    Rate          sdk.Dec   // current commission rate
    MaxRate       sdk.Dec   // maximum commission rate (immutable after creation)
    MaxChangeRate sdk.Dec   // maximum daily rate of change (immutable after creation)
}
```

### Validation Rules

- `MaxRate` must be between 0 and 1 (inclusive).
- `Rate` must be between 0 and `MaxRate`.
- `MaxChangeRate` must be between 0 and `MaxRate`.
- `Rate` must be >= chain-level `MinCommissionRate` (Sei default: 5%).

### Changing Commission (MsgEditValidator)

- `CommissionRate` field is optional; only set it to change.
- New rate cannot be changed more than once per **24 hours**.
- New rate cannot exceed `MaxRate`.
- Rate change (new - old) cannot exceed `MaxChangeRate`.
- `MaxRate` and `MaxChangeRate` are **immutable** after validator creation.

---

## 9. Reward Distribution

### BeginBlock (distribution module)

Each `BeginBlock`, the distribution module allocates fees from the previous block:

1. **Proposer reward**: `feesCollected * (BaseProposerReward + BonusProposerReward * previousFractionVotes)`.
2. **Validator rewards**: remaining fees after proposer reward and community tax, distributed proportionally to voting power.
3. **Community pool**: receives `CommunityTax` fraction plus any rounding dust.

### Per-Validator Allocation

For each validator, rewards are split:
- **Commission**: `reward * validator.CommissionRate` -> validator's accumulated commission
- **Delegator rewards**: `reward * (1 - CommissionRate)` -> shared proportionally among delegators by shares

### Sei Default Distribution Parameters

| Parameter | Default |
|---|---|
| `CommunityTax` | **0%** |
| `BaseProposerReward` | **0%** |
| `BonusProposerReward` | **0%** |

With all three at 0%, all fees are distributed proportionally to voting power with no proposer bonus and no community tax.

---

## 10. Governance Voting

### MsgVote

```
MsgVote {
    ProposalId uint64
    Voter      string       // bech32 account address
    Option     VoteOption   // Yes, No, Abstain, NoWithVeto
}
```

### MsgVoteWeighted

Allows split voting across multiple options with weights summing to 1.0.

### Who Signs

The `Voter` field is an **account address** (`sdk.AccAddress`). For a validator to vote, the validator's **operator account key** signs the transaction. The voter address is the operator's `AccAddress`.

### How Validator Votes Affect Tally

During tally (`Tally()` in `x/gov/keeper/tally.go`):

1. All **bonded** validators are loaded with their bonded tokens and delegator shares.
2. For each vote:
   - If the voter is a validator, their vote is recorded.
   - For each delegation the voter has to a bonded validator, voting power is calculated as `delegationShares * validatorBondedTokens / validatorTotalShares` and added to the corresponding vote option.
   - The validator's delegator deductions are tracked so the validator's own vote only counts for their non-delegated-to-by-voters portion.
3. Validators who vote use their remaining (non-deducted) share power.
4. **Validators who don't vote have NO votes counted** -- their power is simply not tallied (unlike some chains where validator votes inherit to delegators).

### Tally Thresholds (Defaults)

| Parameter | Regular | Expedited |
|---|---|---|
| `Quorum` | **33.4%** | **66.7%** |
| `Threshold` (Yes votes of non-abstain) | **50%** | **66.7%** |
| `VetoThreshold` (NoWithVeto of total) | **33.4%** | **33.4%** |
| `VotingPeriod` | **2 days** | **1 day** |
| `MinDeposit` | **10,000,000** (bond denom) | **20,000,000** |

### Tally Logic

1. If total bonded tokens is zero: proposal fails.
2. If `percentVoting < Quorum`: proposal fails, deposits burned.
3. If everyone abstains (total - abstain == 0): proposal fails.
4. If `NoWithVeto / totalVotingPower > VetoThreshold`: proposal fails, deposits burned.
5. If `Yes / (totalVotingPower - Abstain) > Threshold`: proposal passes.
6. Otherwise: proposal fails.

---

## 11. Validator Set Propagation to Tendermint

### The EndBlock Pipeline

```
Staking EndBlocker (every block)
    |
    v
BlockValidatorUpdates()
    |
    +-> ApplyAndReturnValidatorSetUpdates()
    |       - Iterates validators by descending power
    |       - Bonds/unbonds validators as needed
    |       - Returns []abci.ValidatorUpdate (consensus key + power)
    |
    +-> UnbondAllMatureValidators()
    |       - Completes unbonding for validators past their UnbondingTime
    |
    +-> DequeueAllMatureUBDQueue()
    |       - Returns matured unbonding delegation tokens to delegators
    |
    +-> DequeueAllMatureRedelegationQueue()
            - Completes matured redelegations
```

### ABCI ValidatorUpdate Format

Each update contains:
- `PubKey`: the validator's consensus public key
- `Power`: the validator's new consensus power (0 = remove from Tendermint validator set)

Tendermint receives these updates after `EndBlock` and applies them starting from the **next** block.

### Evidence Flow (Tendermint -> Cosmos SDK)

1. Tendermint detects `DuplicateVoteEvidence` during consensus.
2. Evidence is verified in the evidence pool (`internal/evidence/verify.go`): checks freshness (MaxAge), validates signatures, checks against the validator set at the evidence height.
3. Evidence is included in the next block's `BeginBlock` request as `ByzantineValidators`.
4. The SDK's `x/evidence.BeginBlocker` processes each piece:
   - `DUPLICATE_VOTE` and `LIGHT_CLIENT_ATTACK` are handled as equivocation
   - Validator is slashed, jailed, and tombstoned

---

## 12. Editing a Validator (MsgEditValidator)

```
MsgEditValidator {
    Description      Description      // all fields optional (set "[do-not-modify]" to keep)
    ValidatorAddress string
    CommissionRate   *sdk.Dec         // optional, new commission rate
    MinSelfDelegation *sdk.Int        // optional, new minimum (can only increase)
}
```

- Signed by the **validator operator key**.
- `MinSelfDelegation` can only be **increased**, never decreased.
- If the new `MinSelfDelegation` exceeds the validator's current tokens, the edit is rejected.
- Commission rate changes are subject to the 24-hour cooldown and `MaxChangeRate` limit.

---

## 13. Key Sei-Specific Differences from Standard Cosmos

| Feature | Standard Cosmos SDK | Sei (sei-cosmos v0.3.66) |
|---|---|---|
| `MaxValidators` | 100 | **35** |
| `MinCommissionRate` | 0% | **5%** |
| `SlashFractionDoubleSign` | 5% | **0%** (no slash by default) |
| `SlashFractionDowntime` | 0.01% | **0%** (no slash by default) |
| `SignedBlocksWindow` | 100 | **108,000** (~12 hours at 0.4s blocks) |
| `MinSignedPerWindow` | 50% | **5%** (very lenient) |
| `MaxVotingPowerRatio` | N/A | **Exists** (caps single-validator power as % of total) |
| `MaxVotingPowerEnforcementThreshold` | N/A | **Exists** (minimum total power before ratio is enforced) |
| Distribution defaults | Non-zero proposer rewards + community tax | **All 0%** (pure proportional distribution) |
| Slashing BeginBlocker | Sequential per validator | **Concurrent** (parallel goroutines per validator, sequential writes) |
| Expedited governance | Not in standard SDK | **Supported** (shorter voting period, higher quorum/threshold, higher deposit) |

---

## 14. Complete Transaction Reference

| Transaction | Module | Signer | Purpose |
|---|---|---|---|
| `MsgCreateValidator` | staking | Operator key | Register new validator + self-delegate |
| `MsgEditValidator` | staking | Operator key | Update description, commission, min self-delegation |
| `MsgDelegate` | staking | Delegator key | Delegate tokens to a validator |
| `MsgBeginRedelegate` | staking | Delegator key | Move delegation between validators |
| `MsgUndelegate` | staking | Delegator key | Begin unbonding tokens from a validator |
| `MsgUnjail` | slashing | Operator key | Request unjailing after jail period |
| `MsgVote` | gov | Any account key | Cast governance vote |
| `MsgVoteWeighted` | gov | Any account key | Cast split governance vote |
| `MsgSubmitProposal` | gov | Any account key | Submit governance proposal |
| `MsgDeposit` | gov | Any account key | Add deposit to a proposal |
