---
sip: 2
title: voyager dpos + bft consensus activation
author: Satya Kwok <satya@sentrixchain.com>
status: Final
type: Standards Track
category: Core
created: 2026-05-28
---

## Abstract

SIP-2 documents the activation of **Voyager consensus** on Sentrix Chain — a
move from the pre-Voyager round-robin Proof-of-Authority producer with no
finality, to a stake-weighted DPoS validator set running a 3-phase
Tendermint-style BFT round (propose → prevote → precommit) that finalises a
block on a 2/3+1 stake-weighted quorum. Activation is gated by two compile-time
constants resolved at runtime via `VOYAGER_DPOS_HEIGHT` and `VOYAGER_EVM_HEIGHT`
env vars: below the fork height the chain remains Pioneer (pre-Voyager); at and
above, every block goes through the BFT engine and EVM transactions become
consensus-valid. Mainnet activated 2026-04-25 at `h=579047` for DPoS+BFT and
`h=579060` for EVM. Testnet defaults to height `10` (DPoS) / `752` (EVM).

## Motivation

The pre-Voyager chain ran a single deterministic block producer per height
chosen by a round-robin over a hard-coded authority list. That model had three
structural problems that blocked any ecosystem growth:

1. **No finality** — every block was probabilistically final only after the
   next block referenced its hash, and reorgs could in principle revert any
   non-finalised history. dApps and bridges that need a finality signal had
   nothing concrete to subscribe to.
2. **No permissionless validator entry** — adding or removing a producer was
   a code change to a constant. There was no way for an external operator to
   bond stake and start producing without a chain-repo PR.
3. **No EVM execution** — transactions were native-only (`TokenOp`,
   `StakingOp`, `Transfer`). Solidity contracts could not be deployed.

The fix required three interlocking changes that had to ship as a single
coordinated fork:

- A DPoS validator-set ranked by stake.
- A BFT engine that produces a finality signal for each block.
- An EVM execution path (`revm 37`-based) gated on whether the chain has
  crossed the EVM activation height.

## Specification

### Affected modules

| Crate | Path | Role |
|---|---|---|
| `sentrix-bft` | `crates/sentrix-bft/` | 3-phase BFT round (propose / prevote / precommit). 2/3+1 stake-weighted finality. |
| `sentrix-staking` | `crates/sentrix-staking/` | Stake registry, validator activation, jailing, slashing, reward accrual. |
| `sentrix-evm` | `crates/sentrix-evm/` | `revm 37` adapter, EIP-1559 gas model, precompiles. |
| `sentrix-core` | `crates/sentrix-core/src/block_executor.rs`, `crates/sentrix-core/src/blockchain.rs` | `voyager_mode_for(height)` runtime check, block-apply dispatch, EVM gate. |

### Activation gates

Two independent fork heights, both resolved through
`crates/sentrix-core/src/fork_heights.rs`:

```rust
// Compile-time defaults
const VOYAGER_DPOS_HEIGHT_DEFAULT: u64 = u64::MAX;
const VOYAGER_DPOS_HEIGHT_TESTNET_DEFAULT: u64 = 10;

const VOYAGER_EVM_HEIGHT_DEFAULT: u64 = u64::MAX;
const VOYAGER_EVM_HEIGHT_TESTNET_DEFAULT: u64 = 752;
```

Runtime overrides via env vars `VOYAGER_FORK_HEIGHT` (the DPoS gate) and
`VOYAGER_EVM_HEIGHT`. The two are sequenced — DPoS+BFT activates first, EVM
follows shortly after on the same fork sequence.

Production heights:

| Network | DPoS+BFT (`VOYAGER_FORK_HEIGHT`) | EVM (`VOYAGER_EVM_HEIGHT`) |
|---|---|---|
| Mainnet | `579047` (activated 2026-04-25) | `579060` (activated 2026-04-25) |
| Testnet | `10` (default) | `752` (default) |

### Block-apply dispatch

`block_executor::apply_block_*` reads `self.voyager_mode_for(height)`. Below
the fork the legacy Pioneer apply path runs (round-robin producer check, no
BFT verification). At or above, the block must carry a valid BFT
justification (set of precommits with 2/3+1 stake) and the producer must be
the BFT engine's elected proposer for that `(height, round)`.

EVM transactions (`tx.is_evm_tx()`) are gated separately on
`voyager_mode_for(height) && height >= VOYAGER_EVM_HEIGHT`. Below either gate
an EVM tx is rejected at admission.

### Determinism invariants preserved

- Block hash is unchanged across the fork. State root is still the BSMT root
  over the account database.
- Pre-Voyager block validation rules continue to apply to pre-fork blocks
  retrieved from storage — historical blocks remain decodable.
- The `voyager_activated` boolean on `Blockchain` is in-memory only; it's
  recomputed on every boot from the current tip height vs the fork constants.

## Rationale

**DPoS+BFT over PoS-only or pure PoA**: Sentrix needs both deterministic
finality (BFT contributes this) and permissionless validator entry (DPoS
contributes this). Pure PoA keeps the authority list closed; pure PoS without
BFT keeps the probabilistic-finality problem.

**3-phase Tendermint-style round**: well-understood literature, mature crash
behaviour, and a fixed message-complexity ceiling per round. The simpler
2-phase PBFT variant was rejected because the third phase (precommit) is the
quorum-locking signal that turns "I voted yes" into "I will not vote for any
other block at this height".

**Sequenced EVM activation** (DPoS first, EVM 13 blocks later): the EVM gate
is independent so a future fork could disable EVM without touching consensus
(e.g. emergency response to a revm vulnerability). The 13-block sequencing
on mainnet was a deployment artifact, not protocol-load-bearing.

## Backwards Compatibility

This is a hard fork. Validators must run a binary that knows about
`voyager_activated` and the BFT engine. A pre-Voyager binary continues to
follow the chain only until the fork height, then halts because BFT block
validation fails on every subsequent block.

Migration path for node operators:

1. Upgrade to a Voyager-capable binary (any from the post-`v2.0.x` series).
2. Confirm `VOYAGER_FORK_HEIGHT` env var matches the canonical value
   (`579047` mainnet, `10` testnet — the constants in
   `fork_heights.rs` resolve to these defaults if the env var is absent).
3. At the activation block the validator either produces or signs as part of
   the BFT round; no manual halt-restart required (the binary handles the
   transition).

Migration path for dApp developers: EVM tools (MetaMask, ethers, viem,
hardhat) work from `VOYAGER_EVM_HEIGHT` onward. Native REST + JSON-RPC
`sentrix_*` namespaces continue to work uninterrupted before and after.

Migration path for indexers: the block header gains a BFT justification
field at and above the DPoS fork. Indexers that consumed only the
transaction list and account state are unaffected; indexers tracking
validator participation should start reading the precommit set from the
justification.

## Security Considerations

**New attack surfaces**:

- BFT engine. A bug in `sentrix-bft` can stall consensus by producing
  invalid precommits or rejecting valid ones. Mitigation: extensive
  state-machine unit tests, multi-validator integration harness, and the
  pre-fork bake period observed before the mainnet activation.
- Stake-weighted voting. A validator with majority stake can in theory
  censor or finalise selectively. Mitigation: `MIN_SELF_STAKE` (15,000 SRX)
  + jailing on liveness misses + slashing on double-sign.

**Recovery procedures** if the fork misbehaves on mainnet:

- Halt-all + simul-start with the validators rolled back to the previous
  consensus binary. The pre-Voyager binary will refuse blocks at and above
  the fork height, so the chain stops cleanly rather than corrupting.
- Operator coordination via the canonical chain.db rsync procedure
  documented in the recovery runbooks. Frozen-rsync from the validator with
  the canonical chain.db is the recovery primitive — never whole-dir rsync
  (overwrites `identity/` and `wallets/`).

## Reference Implementation

Implementation is in the chain repo at `sentrix-labs/sentrix` across the
`v2.0.x` release series. The Voyager fork constants and runtime accessors
live at `crates/sentrix-core/src/fork_heights.rs`. The BFT engine is the
`sentrix-bft` crate; the EVM adapter is `sentrix-evm`.

Audit / design docs:

- `audits/bft-signing-fork-design.md` — BFT signing payload spec, including
  the Phase 1 (foundation) helpers.
- `docs/architecture/CONSENSUS.md` — narrative architecture description.

## Copyright

This SIP is dedicated to the public domain under CC0-1.0.
