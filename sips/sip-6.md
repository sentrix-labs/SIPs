---
sip: 6
title: off-trie consensus state commitment (bug a fix)
author: Satya Kwok <satya@sentrixchain.com>
status: Draft
type: Standards Track
category: Core
created: 2026-05-31
---

## Abstract

`pending_rewards`, `total_minted`, liveness counters, and epoch state are consensus-critical but live OUTSIDE the state_root commitment. Mutations happen via `distribute_reward` / `record_block_signatures` / `epoch_manager.record_block` calls that fire AFTER `bc.add_block(blk)` completes — too late for state_root to reflect them. Drift accumulates silently across validators and is undetectable until a transaction CONSUMES the drifted state (ClaimRewards drain, Unjail floor check, AddSelfStake registry mutation). When that happens, vals compute different post-tx state → state_root forks immediately.

This SIP specifies the fork-rule change to commit off-trie state into the state_root commitment so drift is caught at the next block apply.

Three approaches evaluated. **Recommended: F2 (move off-trie state INTO the state trie).** F3 (B3b-style reconciliation system tx) is an operational band-aid; F1 (gate consuming ops) blocks legit users.

## Motivation

The pattern in `bin/sentrix/src/main.rs` FinalizeBlock arms (L1991 self-propose, L2677 peer-propose) and `crates/sentrix-network/src/libp2p_node.rs` apply paths (L1109, L1702, L2014):

```rust
match bc.add_block(blk) {           // ← computes state_root from txs ONLY
    Ok(()) => {
        // POST-add_block — NOT in state_root commitment:
        bc.slashing.record_block_signatures(...);   // liveness counters
        bc.stake_registry.distribute_reward(...);    // pending_rewards
        bc.epoch_manager.record_block(...);          // epoch state
    }
}
```

`state_root` commits to account balances + EVM storage. Off-trie state lives in `stake_registry`, `slashing.liveness`, and `epoch_manager` modules — serialized into MDBX `TABLE_STATE` as a bincode blob, NOT participating in trie hashing.

### Why drift accumulates

Historically per [reward-distribution-flow-audit-2026-04-27.md](https://github.com/sentrix-labs/sentrix/blob/main/audits/reward-distribution-flow-audit-2026-04-27.md): pre-2026-04-27 the libp2p catch-up sync path didn't call the post-add_block bookkeeping AT ALL. Validators that synced via catch-up had different `pending_rewards` than vals that received blocks via gossip. Drift accumulated.

Post-fix, all 5 apply paths call the bookkeeping. Should be deterministic — same `signers` from `block.justification.precommits` produces same `distribute_reward` outputs. But:

- Any one-off bug, env-var quirk, or race that skips bookkeeping on one path silently creates drift
- Drift is **undetectable** by state_root (the only cluster-wide consistency check)
- Drift surfaces ONLY when a tx consumes the drifted state

### Live surface

| Tx | Consumes | Failure mode if drifted |
|---|---|---|
| `StakingOp::ClaimRewards` | `pending_rewards[sender]` | Each val drains different amount → balance diverges → state_root forks |
| `StakingOp::Unjail` | `self_stake >= MIN_SELF_STAKE` + `current_height >= jail_until` | Drifted `self_stake` → different accept/reject decisions per val |
| `StakingOp::AddSelfStake` | implicit (auth `tx.from in registry`) | Less direct, but registry mutations diverge |
| Reward apply per block | distribute_reward outputs | Slow accumulation, eventually surfaces |
| Epoch boundary handlers | `epoch_manager` state + active set recompute | Per-epoch consensus break risk |

### Why this isn't a corner case

Per memory [`project_total_minted_drift_real_v2_2_17`](https://github.com/sentrix-labs/sentrix/blob/main/memory/project_total_minted_drift_real_v2_2_17.md): on 2026-05-26 testnet v2.2.17 boot, all 4 vals had divergent `total_minted` (7 SRX delta val1↔val2). The B3b workaround was a partial heal but the underlying off-trie surface stayed unaddressed.

On 2026-05-31 testnet h=5,817,131: val3's `ClaimRewards` tx surfaced the latent drift — STATE-FP fingerprints diverged immediately post-claim (3-way fork across val1/val2/val4). Recovery required forensic cp from the consensus-aligned validator. See [`project_testnet_cascade_2026_05_31.md`](https://github.com/sentrix-labs/sentrix/blob/main/memory/project_testnet_cascade_2026_05_31.md).

### Why a fix is needed

- Testnet val3 has been stopped since 2026-05-30 cascade because unjail-via-staking-CLI = AddSelfStake + Unjail = re-fork risk
- Mainnet remains off pending the fix
- Any future operator who tries `sentrix staking claim-rewards` or `unjail` on a chain with historical drift will trigger consensus break
- The off-trie state is too useful (rewards accumulator, liveness slashing, epoch tracking) to remove — proper fix must commit it to consensus

## Specification

### Approach F2 — move off-trie state INTO state trie (recommended)

Each off-trie field becomes a trie node, keyed deterministically, so every mutation participates in state_root computation. Drift detected at next block apply: receiver's local recompute of state_root differs from proposer's claim → block rejected.

#### Affected state

| Module | Field | Current location | Proposed trie key |
|---|---|---|---|
| `stake_registry.validators[*]` | `pending_rewards` | `TABLE_STATE` blob | `keccak256("staking:pending_rewards:" + addr)` |
| `stake_registry.validators[*]` | `delegator_rewards` (per delegator) | `TABLE_STATE` blob | `keccak256("staking:delegator_rewards:" + val + ":" + delegator)` |
| `stake_registry` | `total_minted` | `TABLE_STATE` blob | `keccak256("staking:total_minted")` |
| `accounts` | `total_burned` | already in trie via accounts | (no change) |
| `slashing.liveness` | `signed_count`, `missed_count` per val | `TABLE_STATE` blob | `keccak256("liveness:" + addr)` |
| `epoch_manager` | current epoch, end_height, last_recompute_height | `TABLE_STATE` blob | `keccak256("epoch:state")` |

Each mutation now goes through the trie:
```rust
// Old:
val_mut.pending_rewards = val_mut.pending_rewards.saturating_add(commission);

// New:
let key = pending_rewards_trie_key(validator_addr);
let current = trie.get(&key)?.map(decode_u64).unwrap_or(0);
let new_value = current.saturating_add(commission);
trie.insert(key, encode_u64(new_value))?;
```

#### Storage cost

For 21-active-validator network:
- 21 validators × `pending_rewards` = 21 trie entries (~32B value each)
- O(N×M) delegator entries where M = avg delegators per val (low single digits expected post-launch, can grow)
- 21 × `liveness` per val = 21 entries (~16B each: 2 × u64)
- 1 × `total_minted`
- 1 × `epoch state` (~32B)

Total: ~50-100 trie entries vs ~10K accounts trie entries today. Negligible storage impact. Trie depth unchanged.

#### Compute cost

Each mutation: trie get + trie insert. With trie cache (already exists per `sentrix-trie/src/cache.rs`), warm-path reads/writes are O(log N) MDBX page accesses. Per-block overhead: O(active_set × 1) for `distribute_reward` + O(signers × 1) for liveness. At 21 validators: ~42 trie ops per block. Trie path benchmark indicates ~20μs per op warm-cache. **~1ms per block overhead**. Manageable.

#### Migration

Fork-gated by new height `OFF_TRIE_STATE_IN_TRIE_HEIGHT`. Migration block at the fork height:

```rust
// One-time migration at fork activation:
if block.index == OFF_TRIE_STATE_IN_TRIE_HEIGHT {
    // Iterate all validators, copy pending_rewards / liveness / total_minted
    // FROM stake_registry.validators[*] (the bincode blob) INTO trie.
    for (addr, val) in &stake_registry.validators {
        trie.insert(pending_rewards_trie_key(addr), encode_u64(val.pending_rewards))?;
        for (delegator, amount) in &val.delegator_rewards {
            trie.insert(delegator_rewards_trie_key(addr, delegator), encode_u64(*amount))?;
        }
        trie.insert(liveness_trie_key(addr), encode_liveness(&liveness.get_stats(addr)))?;
    }
    trie.insert(total_minted_trie_key(), encode_u64(stake_registry.total_minted))?;
    trie.insert(epoch_state_trie_key(), encode_epoch(&epoch_manager))?;
}
```

**Problem**: drifted chains (testnet right now) have DIFFERENT off-trie values per validator. The migration block would write per-validator values — but each validator writes its OWN local view. State_root mismatch at the migration block itself → consensus fork.

**Solution**: drift reconciliation step BEFORE migration:
- Option M1: pre-migration system tx commits canonical values via consensus (B3b-style). Each val proposes its local values; supermajority winner becomes canonical.
- Option M2: snapshot from a designated "canonical" validator + bootstrap fresh. Loses non-canonical history but guaranteed convergent.
- Option M3: reset to zero. All `pending_rewards` zeroed at fork. Validators lose unclaimed rewards but consensus convergent. Most conservative.

For testnet recovery, M3 is acceptable (rewards are toy). For mainnet, M1 (B3b reconciliation) preferred but requires careful design.

#### Engine + apply path changes

Affected files:
- `crates/sentrix-staking/src/staking.rs` — `distribute_reward`, `add_self_stake`, `register_validator`, `unjail`, `force_unjail` route through trie
- `crates/sentrix-staking/src/slashing.rs` — `record_block_signatures` routes through trie
- `crates/sentrix-staking/src/epoch.rs` — `record_block`, epoch transitions route through trie
- `crates/sentrix-core/src/block_executor.rs` — `ClaimRewards` dispatch reads `pending_rewards` from trie + drains via trie
- `crates/sentrix-trie/src/` — new helper functions for staking-specific keys/encoding

#### Cost summary

- Code change: ~500-1000 LoC across 5 crates
- Migration logic: ~200 LoC + tests
- Reconciliation (M1): ~300 LoC consensus message + verification
- Total: ~1-2k LoC, 2-4 weeks engineering
- Per-block apply overhead: ~1ms (negligible)
- Storage: +50 trie entries per epoch (negligible)

### Approach F3 — B3b-style epoch-boundary reconciliation (alternative)

Periodic reconciliation system tx commits authoritative off-trie state via consensus. Validators run BFT round on the off-trie snapshot; supermajority winner becomes canonical.

#### Mechanism

At every epoch boundary (`EPOCH_LENGTH = 28,800` blocks ≈ 1 day):
- Proposer builds `StakingOp::ReconcileOffTrieState` system tx with:
  - All validators' `pending_rewards`
  - All `delegator_rewards`
  - `total_minted`
  - Liveness counters
  - Epoch state
- Validators verify against their local view — if local matches proposer's, prevote yes; else prevote nil
- On supermajority prevote yes → block applies, all vals force-sync local off-trie state to the committed values
- On supermajority nil → engine timeouts, next round proposer rebuilds with their view. Eventually converges.

#### Trade-off vs F2

| Property | F2 (move into trie) | F3 (B3b reconciliation) |
|---|---|---|
| Drift detection | Every block (state_root mismatch) | Every epoch (28,800 blocks ≈ 1 day) |
| Recovery | Automatic at next apply | Requires reconciliation round to succeed |
| Storage | +50 trie entries per epoch | Same as today |
| Compute | +1ms per block | One large reconciliation tx per epoch |
| Code change | ~1-2k LoC | ~500 LoC |
| Migration complexity | Single migration block + drift reconciliation | First reconciliation block heals existing drift |
| State proof story | clean — pending_rewards in trie, can serve Merkle proofs | requires separate index for proofs |

F3 ships faster but accepts up to 1 epoch of drift before detection. F2 is the architecturally clean answer.

### Approach F1 — gate consuming ops via fork height (alternative)

Add fork gate that disables `ClaimRewards` / `Unjail` / `AddSelfStake` dispatch until F2 or F3 ships. Operator opt-in to re-enable per chain.

#### Trade-off

Pros: ~10 LoC, immediate safety. Cons: blocks legitimate users for months while F2/F3 designed + implemented. Operator can't unjail jailed validators. Effectively pauses chain progress in staking dimension.

### Recommendation

**F2.** Proper architectural fix. Bug A is a recurring class — every off-trie value risks drift forever otherwise. F3 is band-aid that accepts up to 1 epoch of drift between checks. F1 is operational paralysis.

F2 also enables future features that benefit from trie-committed state (Merkle proofs for rewards, cross-chain reward claims, etc).

### Activation plan

1. SIP-6 merged in `Draft` → `Review` after one round of feedback
2. Implementation PR opened against `sentrix-labs/sentrix`. Targets `OFF_TRIE_STATE_IN_TRIE_HEIGHT`. Mainnet default `u64::MAX`, testnet default = `SIP merge time + 100K blocks` (≈ 28 hours of buffer @ 1s)
3. Migration block carries drift reconciliation (option M1 preferred — system tx with consensus-agreed off-trie snapshot)
4. CI: full test pass + new tests for trie-committed mutations + cross-validator agreement
5. Testnet activation via testnet default. **Critical**: testnet currently has unrecovered drift — migration must handle existing 3-way fork state at val3-affected blocks. Likely requires snapshot restore + replay
6. Bake ≥2 weeks (longer than SIP-5 because state migration is heavier)
7. After bake, SIP moves to `Last Call` for 1 week
8. Mainnet activation height chosen during operator halt-all + simul-start window
9. SIP moves to `Final` once mainnet activation has passed without rollback

### Rollback plan

If F2 breaks during testnet bake:

- Pre-fork blocks: unaffected (legacy off-trie path)
- Post-fork blocks: trie-committed path. Rollback = set `OFF_TRIE_STATE_IN_TRIE_HEIGHT_TESTNET_DEFAULT = u64::MAX` env override on all vals, halt-all + simul-start, revert to pre-fork binary
- Migration is irreversible if blocks past the migration height already accepted (trie writes can't be undone without chain rebuild). Bake must catch issues BEFORE mainnet activation.

For mainnet rollback (worst case): same env override, emergency halt-all. Disruptive. Bake discipline is the prevention.

## Backwards compatibility

Pre-fork blocks: zero change. Existing `stake_registry.validators[*].pending_rewards` blob still serialized in `TABLE_STATE`, bookkeeping calls still mutate the blob.

Post-fork blocks: bookkeeping mutations route through trie. The blob in `TABLE_STATE` is left empty / unused for the migrated fields. Read paths must check trie first, fall back to blob for legacy.

Chain history pre-fork is bit-identical. Post-fork chain has additional trie entries reflected in state_root.

## Reference implementation

To be filled by implementation PR. Skeleton:

- New trie key helpers in `crates/sentrix-trie/src/staking_keys.rs`:
  - `pending_rewards_key(addr) -> [u8; 32]`
  - `delegator_rewards_key(val, delegator) -> [u8; 32]`
  - `total_minted_key() -> [u8; 32]`
  - `liveness_key(addr) -> [u8; 32]`
  - `epoch_state_key() -> [u8; 32]`
- New encode/decode helpers (use varint or fixed-width per field)
- `stake_registry.distribute_reward` refactor: read pending_rewards from trie, write back via trie
- `slashing.record_block_signatures` refactor: trie-based liveness counters
- `epoch_manager.record_block` refactor: trie-based epoch state
- `block_executor::apply_block` ClaimRewards arm: trie-based drain
- New migration block logic at `OFF_TRIE_STATE_IN_TRIE_HEIGHT`
- New system tx variant `StakingOp::ReconcileOffTrieState { pending_rewards, delegator_rewards, total_minted, liveness, epoch }` for migration M1
- Cluster-wide testnet recovery procedure for pre-existing drift (snapshot + bootstrap from canonical val)

## Security considerations

- Trie writes increase block apply cost but stay within current per-block time budget at 21-val scale
- Migration block is consensus-critical — every val must produce same trie keys/values from the same off-trie source. Determinism guaranteed by deterministic serialization
- Reconciliation system tx (M1) means proposer's claim of off-trie state is verified by validators against THEIR OWN local view. Adversarial proposer cannot inject fake rewards
- State_root after migration commits off-trie state. Any future divergence detected at first post-migration block. No regression of safety
- Existing drift on testnet must be addressed BEFORE migration: either reset (M3) or canonical-val snapshot (M2). Migration won't auto-heal drift, only catch FUTURE drift

## Interaction with SIP-5

SIP-5 (Bug B fix, block hash + state_root consistency) and SIP-6 (Bug A fix, off-trie state) are independent in mechanism but related in motivation. Both surfaced from 2026-05-31 testnet cascade.

Activation order matters:
- **SIP-5 before SIP-6**: cleaner. SIP-5's speculative-apply requires state_root determinism, which SIP-6 strengthens by committing more state. Without SIP-6, speculative apply's `pending_rewards` mutations during apply also affect state_root → SIP-5's hash claim has to include those updates. Awkward but workable.
- **SIP-6 before SIP-5**: requires SIP-5's bug B latency to continue. Existing `block.hash != j.block_hash` inconsistency persists.
- **Both in same fork window**: cleanest. One coordinated operator activation, one bake period.

Recommend: ship both via same `BLOCK_HASH_WITH_REAL_STATE_ROOT_HEIGHT == OFF_TRIE_STATE_IN_TRIE_HEIGHT` height. Operator activates both atomically.

## Open questions

1. Reconciliation (M1) requires consensus-agreed off-trie snapshot. What's the verification rule — bit-equal blob, per-field equality, or stake-weighted majority per field?
2. Existing testnet drift: snapshot-from-canonical (M2) loses some validators' history; reset-to-zero (M3) loses all pending rewards. Operator preference?
3. Should `epoch_manager` state move to trie too, or stay separate (it's not consensus-critical in same way as rewards)?
4. Trie size impact at 1000+ validators (future scale) — need benchmarks

## References

- Issue [#750](https://github.com/sentrix-labs/sentrix/issues/750) — Bug A tracking
- Issue [#751](https://github.com/sentrix-labs/sentrix/issues/751) — Bug B (related, SIP-5)
- Memory `project_bug_a_off_trie_state_drift.md` — internal trace
- Memory `project_total_minted_drift_real_v2_2_17.md` — historical drift incident
- Memory `project_testnet_cascade_2026_05_31.md` — surfacing incident
- Audit: `reward-distribution-flow-audit-2026-04-27.md`

## Copyright

This SIP is licensed under MIT-OR-Apache-2.0 (matching the SIPs repo).
