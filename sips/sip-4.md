---
sip: 4
title: reward distribution v2 — treasury escrow and ClaimRewards
author: Satya Kwok <satya@sentrixchain.com>
status: Final
type: Standards Track
category: Core
created: 2026-05-28
requires: 2
---

## Abstract

SIP-4 specifies the reward distribution v2 fork: the per-block validator
reward stops being credited directly to the block producer's address, and is
instead minted into a protocol treasury address with a per-validator
accumulator on the stake registry. Validators (and their delegators)
withdraw accrued rewards by submitting a `StakingOp::ClaimRewards`
transaction, which the apply path dispatches by draining the accumulator
and transferring SRX from the treasury escrow to the claimer. Activation is
gated by `VOYAGER_REWARD_V2_HEIGHT`; mainnet activated at `h=590100` on
2026-04-25, testnet at `h=100`.

## Motivation

The v1 reward path credited the block reward straight into the producer's
account balance at apply time. That model had three problems:

1. **No commission split**. A validator with delegators received the entire
   block reward; the commission split between validator and delegators had
   to happen off-chain, breaking the chain's role as the source of truth
   for stake economics.
2. **Spam on the supply curve**. Every block triggered an account balance
   write on the producer's address, even when the producer was not actively
   claiming. Indexers tracking supply distribution saw constant churn that
   wasn't economically meaningful (the validator wasn't moving the
   balance, just receiving the reward).
3. **No explicit claim moment**. There was no transaction users could submit
   to mark "I'm taking my rewards now" — accruals and claims were
   indistinguishable on the chain history.

The v2 fork separates accrual (per-block, into a protocol-owned escrow with
a per-validator accumulator) from claim (explicit `ClaimRewards` tx, which
splits between validator commission and delegator pool by the configured
commission rate, then transfers SRX out of the escrow).

## Specification

### Affected components

| Component | Path | Change |
|---|---|---|
| Coinbase distribution | `crates/sentrix-core/src/block_executor.rs` | Post-fork: coinbase mints into `PROTOCOL_TREASURY` instead of the producer's address. |
| Stake registry accumulator | `crates/sentrix-staking/src/staking.rs` | New per-validator + per-delegator `pending_rewards` accumulator. |
| ClaimRewards apply path | `crates/sentrix-core/src/block_executor.rs` | New `StakingOp::ClaimRewards` dispatch. Pre-fork: rejected at admission. Post-fork: drains accumulator + transfers SRX. |
| `take_delegator_rewards` accessor | `crates/sentrix-staking/src/staking.rs:1083` | Drains a single delegator's accumulator atomically; used by the ClaimRewards apply path. |

### Treasury address

`PROTOCOL_TREASURY` is a reserved system address (declared in
`crates/sentrix-core/src/address.rs`). It holds the entire post-fork reward
float between accrual and claim. Total balance at any height equals
`Σ (pending_rewards over all validators and delegators)`.

### Activation gate

```rust
const VOYAGER_REWARD_V2_HEIGHT_DEFAULT: u64 = u64::MAX;
const VOYAGER_REWARD_V2_HEIGHT_TESTNET_DEFAULT: u64 = 100;
```

Runtime override via `VOYAGER_REWARD_V2_HEIGHT` env var.

Production heights:

| Network | `VOYAGER_REWARD_V2_HEIGHT` |
|---|---|
| Mainnet | `590100` (activated 2026-04-25) |
| Testnet | `100` (default) |

### Per-block apply (post-fork)

For every finalised block at `height >= VOYAGER_REWARD_V2_HEIGHT`:

1. Compute `block_reward = get_block_reward()` (fork-gated; SIP-3 covers the
   halving schedule).
2. Add `block_reward` to `total_minted`.
3. Credit `PROTOCOL_TREASURY` with `block_reward`.
4. Accrue `block_reward` against the producer's validator accumulator,
   split between validator-self and delegators per the validator's
   commission rate at the time of the block.

### ClaimRewards apply path

When a `StakingOp::ClaimRewards { validator, claimer }` transaction is
applied:

1. **Pre-fork**: tx is rejected with `consensus_invalid_pre_fork`. No
   accumulator drain. No transfer.
2. **Post-fork**:
   - Drain `take_delegator_rewards(claimer)` from the validator's
     accumulator (validator-self drain uses the same accessor with the
     validator's own address as the claimer).
   - Transfer the drained amount from `PROTOCOL_TREASURY` to `claimer`'s
     address.
   - Emit a `RewardsClaimed { validator, claimer, amount }` event for
     indexer consumption.

### Determinism invariants preserved

- `total_minted` advances exactly `block_reward` per block at and above the
  fork — same accrual rate as pre-fork.
- `PROTOCOL_TREASURY` balance equals the sum of all pending accumulators at
  any height (verified by `total_minted` accounting).
- ClaimRewards is idempotent on the same `(validator, claimer, accumulator
  snapshot)` — draining a zero accumulator is a no-op.

## Rationale

**Treasury escrow over direct-credit**: separates economic accrual (which
must remain deterministic per block) from claim (which is user-initiated
and irregular). Lets indexers track "this is what's payable" separately
from "this has actually been paid out".

**Per-validator accumulator on the stake registry**: keeps the data
structure colocated with stake itself. Avoids a parallel ledger that has to
stay in sync. The validator's `commission_rate` at the time of each block
is captured in the per-block accrual step, so a commission-rate change
doesn't retroactively rewrite pending rewards.

**Explicit claim transaction over auto-distribute**: auto-distribute at the
end of each epoch (or each N blocks) would push the chain into a regular
supply-state-write pattern that defeats the indexer-load motivation. The
explicit claim tx lets the user decide when they want the on-chain event,
and lets the chain stay quiet for inactive validators / delegators.

## Backwards Compatibility

Hard fork at `VOYAGER_REWARD_V2_HEIGHT`. Pre-fork blocks remain decodable;
pre-fork rewards already credited to producer addresses stay on those
addresses. The fork only affects rewards accrued *at and after* the fork
height.

Migration path for validators:

1. Upgrade to a reward-v2-capable binary.
2. Set `VOYAGER_REWARD_V2_HEIGHT` env var to match the canonical height for
   the network. Mainnet `590100`, testnet `100` (compile-time default).
3. After the fork, the validator's wallet stops receiving direct block
   rewards. Submit `StakingOp::ClaimRewards` periodically (or on-demand) to
   move accrued rewards from the treasury escrow into the validator wallet.

Migration path for delegators:

- Pre-fork delegators received rewards via off-chain channels (validator
  side-payments). Post-fork delegators submit `ClaimRewards` themselves
  against the validator they delegated to. The accumulator tracks
  delegator-side accruals separately so the validator can't claim them.

Migration path for indexers:

- `/staking/validators` gains a `pending_rewards` field per validator.
- `RewardsClaimed` events become a new event class — see SIP-supported
  WebSocket subscription for the stream.
- Total supply tracking should sum `PROTOCOL_TREASURY` balance into
  "circulating-pending" if the indexer wants to expose unclaimed-yet
  rewards.

## Security Considerations

**Treasury solvency invariant**: at any height,
`PROTOCOL_TREASURY.balance >= Σ pending_rewards`. A bug that lets a claim
exceed the accumulator would withdraw from another validator's pool. The
invariant is enforced by the `take_delegator_rewards` accessor draining
exactly the accumulator value (and zero-clamping on empty).

**Replay protection**: `ClaimRewards` carries the standard transaction
nonce on the claimer's account. Replaying a ClaimRewards tx after the
accumulator has been drained becomes a no-op (zero-balance drain) — the
tx still consumes the nonce and pays gas, but transfers zero.

**Fork-moment ambiguity**: the first block at exactly
`VOYAGER_REWARD_V2_HEIGHT` mints into the treasury. The block at
`height - 1` is still v1 (direct credit). No reward is double-counted at
the boundary because the height check is exclusive on the v1 side.

**Recovery procedures** if a buggy ClaimRewards drains incorrectly:

- The accumulator state lives in the stake registry, which is part of the
  state root. A faulty drain is therefore visible in state-divergence
  diffs across validators.
- Recovery via chain.db rsync from the canonical validator (the standard
  state-recovery procedure). The transfer side of the bad claim is harder
  to reverse — would need an explicit corrective tx if the bad transfer
  leaves the treasury.

## Reference Implementation

The fork shipped in the `v2.1.x` release series. Treasury escrow logic and
ClaimRewards apply path live in `crates/sentrix-core/src/block_executor.rs`;
the accumulator and `take_delegator_rewards` accessor in
`crates/sentrix-staking/src/staking.rs`. Test coverage at
`crates/sentrix-staking/src/staking.rs:1228+` (Phase D regression suite).

Public documentation:

- `docs/tokenomics/STAKING.md` — staking + delegation + reward mechanics.
- `docs/tokenomics/REWARD_ESCROW.md` — treasury escrow design.

## Copyright

This SIP is dedicated to the public domain under CC0-1.0.
