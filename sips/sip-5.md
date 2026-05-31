---
sip: 5
title: block hash + state root commitment consistency (bug b fix)
author: Satya Kwok <satya@sentrixchain.com>
status: Draft
type: Standards Track
category: Core
created: 2026-05-31
---

## Abstract

A latent consensus bug in Sentrix's block-apply path produces stored blocks where `block.hash != block.justification.block_hash`. The bug surfaced on 2026-05-31 testnet h=5,817,132 after [Bug A](https://github.com/sentrix-labs/sentrix/issues/750) caused state divergence at h=5,817,131. This SIP specifies the fork-rule change to make block hash deterministically equal across a block's lifecycle (pre-apply, post-apply, broadcast, replay).

Two architectural approaches are evaluated. **Recommended: B1 (speculative apply at propose time).** The B4 alternative (drop state_root from `Block::calculate_hash`) is cheaper to implement but downgrades safety by removing state_root from the hash chain.

## Motivation

`crates/sentrix-core/src/block_executor.rs:1561-1563` does:

```rust
// Self-produced: stamp and recompute hash (V7-C-01).
last.state_root = Some(computed_root);
last.hash = last.calculate_hash();
```

Sequence at the proposer:

1. `Block::new(...)` constructs the block with `state_root=None` and `hash=H1` calculated using `state_root_hex="0".repeat(64)` placeholder
2. Proposer broadcasts proposal with `block_hash=H1`
3. Other validators prevote + precommit on `H1`
4. Engine collects supermajority for `H1` → emits `BftAction::FinalizeBlock(H1, justification refs H1)`
5. Validator-loop's hash-mismatch guard passes (`stashed.hash == action.block_hash`, both `H1`)
6. `bc.add_block(blk)` runs apply, computes `computed_root`
7. **L1562**: `last.state_root = Some(computed_root)`
8. **L1563**: `last.hash = last.calculate_hash()` — recomputed with new `state_root` → **`H2 ≠ H1`**
9. `justification.block_hash` and `precommits[*].block_hash` stay at `H1` (not updated)

The stored block has internally-inconsistent `block.hash=H2` and `justification.block_hash=H1`.

### Why the bug is latent

Chain progression is unaffected because:

- Next block's `prev_hash` references `H2` consistently — chain links work
- `STRICT_JUSTIFICATION_HEIGHT` defaults to `u64::MAX`; strict-mode signature recovery against `j.block_hash=H1` is never run
- Pre-fork justification check sums `stake_weight` only — passes regardless of hash relationship
- Validator signatures in `precommits` were signed for `H1`; recovery against `j.block_hash=H1` would succeed under strict mode

The cluster has been running with this bug since the state_root v2 fork shipped. PR [#752](https://github.com/sentrix-labs/sentrix/pull/752) (v2.2.23) added a receiver-side `block.hash == j.block_hash` check inside the existing `strict_justification` fork gate, which would block every future block once activated. Activating strict-justification before this SIP's producer-side fix lands would halt the chain.

### Why a fix is needed now

- Bug B blocks safe activation of `STRICT_JUSTIFICATION_HEIGHT`, which is itself defense-in-depth against [Halt #9](https://github.com/sentrix-labs/sentrix/audits/halt-9-strict-justification.md) drift attacks.
- Until fixed, every stored block carries a hash mismatch — auditors examining the chain see broken invariants.
- Any future code that asserts `block.hash == block.justification.block_hash` would have to special-case all historical blocks.
- The 2026-05-31 incident showed that paired with Bug A, the inconsistency can produce blocks that one cluster of validators stores and another rejects (`invalid previous hash`).

## Specification

### Approach B1 — speculative apply at propose time (recommended)

The proposer pre-applies the block locally to compute `state_root`, includes it in the proposal, and validators verify `state_root` before voting. Everyone agrees on the hash *with* `state_root` from the moment the proposal is broadcast.

#### Proposer-side changes

`Block::new(...)` retains the speculative-apply hash semantics:
- `state_root: None` initially, `hash: H1` (with placeholder)

After `Block::new`, before broadcast:
```rust
// New helper: stamp_speculative_state_root
let computed_root = bc.speculative_apply_for_state_root(&block)?;
block.state_root = Some(computed_root);
block.hash = block.calculate_hash();  // ← now H2 with real state_root
```

The proposal carries `block_hash=H2`. Engine tracks `H2`. Justification refers to `H2`. Stored block's hash field stays `H2` because `add_block` doesn't need to recompute (already at final value).

#### Validator-side changes

In `BftMessage::Propose` handler at main.rs L2540+, after deserializing the proposed Block:

```rust
// 1. Verify proposer's block.hash claim before any state work.
//    Recompute hash from received fields; reject if proposer's claim
//    diverges. This closes the Bug C class at the spec level —
//    validators MUST NOT trust the wire-level block.hash without
//    locally recomputing.
let local_computed_hash = block.calculate_hash();
if local_computed_hash != block.hash {
    tracing::warn!(
        claimed = %hex(block.hash),
        local = %hex(local_computed_hash),
        "proposer block.hash claim diverges from local recomputation — rejecting"
    );
    return;  // don't prevote
}

// 2. Verify proposer's state_root claim. Missing state_root is a
//    protocol violation post-fork; reject rather than unwrap.
let claimed_root = match block.state_root {
    Some(r) => r,
    None => {
        tracing::warn!("post-fork block missing state_root — rejecting");
        return;
    }
};
let local_computed_root = bc_read.speculative_apply_for_state_root(&block)?;
if local_computed_root != claimed_root {
    tracing::warn!("proposer state_root claim diverges from local — rejecting");
    return;  // don't prevote
}
// hash + state_root agreed → safe to prevote on H2.
```

If `block.calculate_hash()` or `speculative_apply_for_state_root` returns different result on validator vs proposer, the proposal is rejected. Cluster won't reach supermajority on a block with disputed hash or state_root.

#### add_block_impl changes

Remove the `last.state_root = Some(computed_root); last.hash = last.calculate_hash()` recompute at L1561-1563 (and sibling sites L1608, L1649, L1654 if similar). The block already has `state_root=Some(R)` and `hash=H2` set by the proposer. `add_block` validates without `unwrap`:

```rust
let claimed_root = block.state_root.ok_or(BlockError::MissingStateRoot)?;
if computed_root != claimed_root {
    return Err(BlockError::StateRootMismatch { computed: computed_root, claimed: claimed_root });
}
```

Missing `state_root` on a post-fork block is a protocol violation and rejected; mismatch is also rejected. No `unwrap` in the normative path.

#### Engine changes

None directly — engine still tracks whatever hash the proposer broadcast. The hash is now `H2` (with state_root), justification refs `H2`. Internal logic unchanged.

#### Cost

- Proposer does block apply TWICE: once for speculative state_root, once for actual chain advance. ~2x apply CPU on proposer.
- Validators do speculative-apply once on receipt + actual apply once on commit. ~2x apply CPU on validators when this is the proposer's block.
- For 21-validator network at 1s blocks, 2x apply is ~0.5s extra per block → bumps target block time. Manageable; matches Tendermint cost.

#### Migration

Fork-gated by new height `BLOCK_HASH_WITH_REAL_STATE_ROOT_HEIGHT`:
- Mainnet default: `u64::MAX` (disabled, pending operator activation window)
- Testnet default: `<current_height + 50_000>` at SIP merge time

**Activation predicate (canonical, used everywhere in this SIP):**

```
is_post_fork(block) := block.height >= BLOCK_HASH_WITH_REAL_STATE_ROOT_HEIGHT
```

The boundary is **`>=`** (inclusive), not `>`. The exact block at the fork height is the FIRST post-fork block. All references in this SIP to "pre-fork" mean `block.height < BLOCK_HASH_WITH_REAL_STATE_ROOT_HEIGHT`; "post-fork" means `block.height >= BLOCK_HASH_WITH_REAL_STATE_ROOT_HEIGHT`. Implementation MUST use this single predicate in every gated branch (proposer flow, validator flow, `add_block_impl`) to prevent off-by-one consensus splits.

Pre-fork blocks: proposer doesn't pre-apply, hash stays at `H1`-with-placeholder, recompute step at L1563 still runs → legacy behavior preserved bit-identically.

Post-fork blocks: proposer pre-applies, hash is `H2`-with-real-root, recompute step skipped → consistent hash across lifecycle.

### Approach B4 — drop state_root from block hash (alternative)

`Block::calculate_hash()` excludes `state_root_hex` from the payload. Block hash is computed only from `index + previous_hash + merkle_root + timestamp + validator`. `state_root` becomes metadata, verified separately at apply time via the existing `received_root != computed_root` check.

#### Trade-off vs B1

| Property | B1 (speculative apply) | B4 (drop state_root from hash) |
|---|---|---|
| State_root in hash chain | yes (unchanged from current) | **no — safety downgrade** |
| Single block hash across lifecycle | yes | yes |
| Justification consistency | yes | yes |
| Implementation complexity | higher (proposer flow + speculative apply API) | lower (single line change in calculate_hash + delete recompute) |
| Proposer compute cost | 2x apply | 1x apply (unchanged) |
| Backwards compat | fork-gated | fork-gated |
| Auditor story | "block hash commits to all consensus state including state_root" — clean | "block hash commits to header + txs, state_root verified separately via field comparison" — less clean |

B4 loses an important property: under B1, an attacker substituting a different state_root would change `block.hash`, breaking the chain link to next block. Under B4, the chain link only depends on header + txs; state_root drift would silently persist if the receiver's separate check has a bug.

The receiver-side state_root check at `block_executor.rs:1567` is currently the only line that catches state_root divergence under B4. A single check is sufficient when correct, but loses the redundancy of having it also commit via the hash chain.

### Recommendation

**B1**, accepting the 2x apply cost. State_root in hash chain is a safety property worth preserving. Tendermint chains pay this cost and Sentrix should match.

### Activation plan

1. SIP-5 merged in `Draft` → `Review` status after one round of feedback
2. Implementation PR opened against `sentrix-labs/sentrix`. Targets new fork height `BLOCK_HASH_WITH_REAL_STATE_ROOT_HEIGHT`. Mainnet default `u64::MAX`, testnet default `<live testnet height + 50_000>` (≈ 14 hours of buffer @ 1s block time).
3. CI: full test pass + new tests for speculative-apply path + cross-validator agreement on `state_root` claim
4. Testnet activation via testnet default. Bake ≥1 week with apply_watchdog observing.
5. After bake, SIP moves to `Last Call` for one week
6. Mainnet activation height chosen during operator halt-all + simul-start window. Update mainnet default in code, ship as separate small PR.
7. SIP moves to `Final` once mainnet activation height has passed and no rollback.

### Rollback plan

If B1 introduces a regression:

- Pre-fork blocks continue using legacy path (untouched)
- Post-fork blocks use B1 path — rollback means halting cluster, reverting binary to pre-B1 version, and never reaching the fork height again
- Because the fork gate is height-based, vals booted on legacy binary at post-fork height would reject the chain (different `add_block_impl` logic). Recovery requires every val to roll back binary in lockstep

In practice: if B1 breaks badly during testnet bake, **set `BLOCK_HASH_WITH_REAL_STATE_ROOT_HEIGHT_TESTNET_DEFAULT = u64::MAX` via env override** on all vals, halt-all + simul-start. Resume legacy behavior.

For mainnet rollback (worst case): same env override on all mainnet vals + emergency halt-all. Doable but disruptive.

## Backwards compatibility

Pre-fork blocks: zero change to hash computation, recompute step at L1563 still runs. Bit-identical chain history preserved.

Post-fork blocks: new hash computation includes real state_root from proposer's speculative apply. Validator-loop's hash-mismatch guard still fires correctly (`stashed.hash == action.block_hash`, both H2 now).

Justification refs match block.hash post-fork. Strict_justification fork can then be activated safely.

## Reference implementation

To be filled in by implementation PR. Skeleton:

- New API: `Blockchain::speculative_apply_for_state_root(&block) -> SentrixResult<[u8; 32]>` that clones the trie + applies block txs + returns root WITHOUT mutating chain state. Cleanup must release the cloned trie immediately to bound memory.
- New fork: `BLOCK_HASH_WITH_REAL_STATE_ROOT_HEIGHT` in `crates/sentrix-core/src/fork_heights.rs`
- Proposer path in `bin/sentrix/src/main.rs` ProposeBlock + skip-round arms: call speculative apply, set `block.state_root` and recompute `block.hash` before broadcasting Proposal
- Validator path: in `BftMessage::Propose` handler, verify proposer's claimed `state_root` matches local speculative apply
- `add_block_impl` at L1561-1563: skip recompute when block already has `state_root=Some(...)` from proposer's claim (post-fork blocks); keep legacy recompute for pre-fork

## Security considerations

- Speculative apply uses the SAME trie state and tx ordering as actual apply. Determinism guaranteed by Sentrix's deterministic execution model. Two applies of the same block by the same validator produce identical state_root.
- Validators rejecting proposals with wrong state_root claim does not weaken liveness — proposer's local apply produces correct state_root, validators' local applies produce same state_root, agreement is trivial.
- If a Byzantine proposer claims wrong state_root, validators reject the proposal and engine times out to next round. Cost: one round of liveness, no safety violation.
- A bug in `speculative_apply_for_state_root` that produces different result than the real `apply` would cause cluster split. Test coverage must include determinism tests across diverse tx workloads (EVM, native transfers, staking ops, claim-rewards, jail/unjail).

## Open questions

1. Should speculative apply also commit `pending_rewards` / `liveness` / `total_minted` mutations (the "Bug A" off-trie state) into a transient calculation for state_root inclusion? This SIP focuses on B1 only; integrating Bug A fix (#750) into the same fork is a related but separate decision.
2. Memory cost of speculative apply on validators (cloned trie per proposal) needs measurement at testnet scale.
3. Cache speculative-apply results so commit-time doesn't redo the work? Optimization for later — initial implementation can do 2x apply.

## References

- Issue [#751](https://github.com/sentrix-labs/sentrix/issues/751) — Bug B tracking
- Issue [#750](https://github.com/sentrix-labs/sentrix/issues/750) — Bug A (related, separate SIP needed)
- PR [#752](https://github.com/sentrix-labs/sentrix/pull/752) — v2.2.23 receiver-side defense check (gated by strict_justification fork)
- Memory `project_bug_b_block_hash_recompute.md` — internal trace
- Memory `project_testnet_cascade_2026_05_31.md` — incident timeline
- Tendermint ABCI++ — speculative execution reference design

## Copyright

This SIP is licensed under MIT-OR-Apache-2.0 (matching the SIPs repo).
