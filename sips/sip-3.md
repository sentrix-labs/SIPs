---
sip: 3
title: tokenomics v2 — 315M cap and 4-year halving
author: Satya Kwok <satya@sentrixchain.com>
status: Final
type: Standards Track
category: Core
created: 2026-05-28
requires: 2
---

## Abstract

SIP-3 specifies the tokenomics v2 fork: a one-shot fork-height transition
from the v1 emission schedule (210M SRX hard cap, 42M-block halving interval
≈ 1.33 years at 1-second blocks) to the v2 schedule (315M SRX hard cap, 126M-
block halving interval ≈ 4 years, BTC-parity). Block reward stays at `1
SRX` per block on both sides of the fork. Premine remains `63,000,000 SRX`
in absolute terms — its ratio to the cap improves from 30% (of 210M) to 20%
(of 315M) without any new mint. Activation is gated by `TOKENOMICS_V2_HEIGHT`;
mainnet activated at `h=640800` on 2026-04-26, testnet at `h=381651`.

## Motivation

The v1 schedule had two problems that compounded over time:

1. **Halving cadence too aggressive**. A 42M-block halving at 1-second blocks
   put the first halving at ~year 1.33. That schedule front-loads emission so
   much that long-term validator incentives drop sharply before the network
   has time to mature. BTC-parity (4-year halving) is the empirically
   validated cadence for a chain that wants validator rewards to track the
   network's adoption curve rather than outrun it.
2. **Cap too low for the eventual address space**. The 210M cap was a
   placeholder. Sentrix's address-space + block-time profile supports a
   larger circulating supply without hyperinflation, and ecosystem partners
   asked for a clearer growth ceiling before integrating.

Doing both as a single fork keeps the change set small and the activation
moment unambiguous — the validator either runs the v2 schedule or it doesn't,
no per-parameter ambiguity.

## Specification

### Constants (compile-time, in `crates/sentrix-core/src/tokenomics.rs`)

```rust
/// V1 cap — 210M SRX. Used for blocks below TOKENOMICS_V2_HEIGHT.
pub const MAX_SUPPLY: u64 = 210_000_000 * 100_000_000;       // sentri (10^8 per SRX)

/// V2 cap — 315M SRX. Used at and above TOKENOMICS_V2_HEIGHT.
pub const MAX_SUPPLY_V2: u64 = 315_000_000 * 100_000_000;

/// Block reward. Unchanged across the fork — 1 SRX per block.
pub const BLOCK_REWARD: u64 = 100_000_000;                   // 1 SRX in sentri

/// V1 halving interval — 42M blocks (~1.33y at 1s blocks).
pub const HALVING_INTERVAL: u64 = 42_000_000;

/// V2 halving interval — 126M blocks (~4y at 1s blocks, BTC-parity).
pub const HALVING_INTERVAL_V2: u64 = 126_000_000;
```

Premine constant lives separately in `crates/sentrix-core/src/address.rs`:

```rust
pub const TOTAL_PREMINE: u64 = 63_000_000 * 100_000_000;     // 63M SRX in sentri
```

### Fork-gated accessors

`Blockchain::max_supply_for(height)` and `halving_interval_for(height)`
dispatch on `is_tokenomics_v2_height(height)`:

```rust
pub fn max_supply_for(height: u64) -> u64 {
    if Blockchain::is_tokenomics_v2_height(height) {
        MAX_SUPPLY_V2
    } else {
        MAX_SUPPLY
    }
}

pub fn halving_interval_for(height: u64) -> u64 {
    if Blockchain::is_tokenomics_v2_height(height) {
        HALVING_INTERVAL_V2
    } else {
        HALVING_INTERVAL
    }
}
```

### Halving-count accumulation across the fork

Halvings accumulated before the fork (against the 42M interval) are
**preserved** at the fork boundary. Post-fork halvings count against the
126M interval starting from the fork height. There is no jump-up in block
reward at the fork moment — the reward at fork height equals the reward at
fork height minus one, so no producer can see a sudden change.

### Activation gate

```rust
const TOKENOMICS_V2_HEIGHT_DEFAULT: u64 = u64::MAX;
const TOKENOMICS_V2_HEIGHT_TESTNET_DEFAULT: u64 = 381_651;
```

Runtime override via `TOKENOMICS_V2_HEIGHT` env var.

Production heights:

| Network | `TOKENOMICS_V2_HEIGHT` |
|---|---|
| Mainnet | `640800` (activated 2026-04-26) |
| Testnet | `381651` (activated earlier on the testnet bake) |

### Premine accounting

Premine remains `63,000,000 SRX` in absolute terms. The percentage
representation changed because the denominator (cap) grew:

| Field | V1 (pre-fork) | V2 (post-fork) |
|---|---|---|
| Cap | 210,000,000 SRX | 315,000,000 SRX |
| Premine (absolute) | 63,000,000 SRX | 63,000,000 SRX |
| Premine (% of cap) | 30% | 20% |
| Halving interval | 42,000,000 blocks | 126,000,000 blocks |
| Time per halving | ~1.33 years | ~4 years (BTC-parity) |

No mint event at the fork. No address balance changes. Premine recipient
addresses (founder, ecosystem, early-validator, reserve) are unchanged and
hold the same balances as they did pre-fork.

## Rationale

**Why 315M cap and not another number**: 315M = 210M × 1.5. The original 210M
was already a placeholder; growing by 50% balanced "headroom for the curve"
against "minimal disruption to existing holder expectations". A doubling to
420M was considered and rejected as too aggressive a denominator change.

**Why 126M halving interval and not, say, 100M or 168M**: BTC parity. A 126M
block × 1-second target = 4 years, matching Bitcoin's halving cadence
(210k blocks × 10-min target ≈ 4 years). Empirically validated cadence.

**Why preserve block reward at 1 SRX**: changing reward at the fork moment
would create a visible spike or dip that producers and watchers would
immediately notice. The interval-only change keeps the per-block reward
amount continuous across the fork.

**Why preserve premine absolute amount**: any modification (burn portion,
re-vest, etc.) would have been a separate decision with its own coordination
cost. v2 deliberately keeps premine untouched so the cap change is the only
parameter under discussion.

## Backwards Compatibility

Hard fork. Validators running a pre-tokenomics-v2 binary will compute the
wrong block reward and wrong max-supply check past the fork height,
diverging from the canonical chain.

Migration path for validators:

1. Upgrade to a tokenomics-v2-capable binary (any from the `v2.1.38+` series,
   verified by PR #337 landing).
2. Confirm `TOKENOMICS_V2_HEIGHT` env var matches the canonical value
   (`640800` mainnet, `381651` testnet) or is unset (compile-time default
   handles testnet, env var required for mainnet).
3. No state migration. No mint event. Block-apply continues normally across
   the fork height.

Migration path for explorers / supply trackers:

- `/chain/info` `max_supply` field flips from 210M to 315M at the fork
  block. Polling consumers should expect the change and rebase their
  ratios.
- Halving countdown displays based on the v1 interval will look "wrong"
  post-fork; update to read `halving_interval_for(current_height)` rather
  than hardcoding 42M.

## Security Considerations

**Determinism**: the fork-gated accessors are pure functions of `height`.
Two validators running the same binary at the same height MUST compute the
same `max_supply_for` and `halving_interval_for`. Any drift = state
divergence.

**Discovery during deploy**: the initial mainnet binary at fork activation
was a pre-PR #337 build (consensus path correct but the RPC display layer
still reporting 210M cap). Resolved on 2026-04-26 evening via halt-all +
simultaneous binary swap to `v2.1.39 + #337` at ~`h=646200`. Lesson:
display-layer drift after a fork is a known class — verify both
consensus-side and presentation-side reports cross the fork together.

**No mint event**: by construction. No address balance is touched at the
fork. No `MAX_SUPPLY` invariant guard fires at the fork boundary because the
v2 cap is strictly larger than the v1 cap and `total_minted` is well below
either.

## Reference Implementation

The fork shipped in PR #337 (`sentrix-labs/sentrix`), part of the `v2.1.38+`
release series. The constants and fork-gated accessors live in
`crates/sentrix-core/src/tokenomics.rs`; the activation gate is in
`crates/sentrix-core/src/fork_heights.rs`. The end-to-end verification trace
across the mainnet fork crossing is captured in the operator runbook (not
public).

Public documentation:

- `docs/tokenomics/SRX.md` — SRX coin spec.
- `docs/tokenomics/OVERVIEW.md` — tokenomics overview.

## Copyright

This SIP is dedicated to the public domain under CC0-1.0.
