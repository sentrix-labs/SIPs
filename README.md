# Sentrix Improvement Proposals (SIPs)

Design changes, fork specifications, and protocol enhancements for [Sentrix Chain](https://github.com/sentrix-labs/sentrix).

A SIP captures *what* is being changed, *why* it is worth changing, and *how* the change ships — at the level of detail another implementer or auditor needs to reason about it. Day-to-day code changes go in regular pull requests against the chain repo; SIPs are reserved for decisions that require ecosystem-wide alignment (consensus rules, fork heights, economic constants, wire-protocol versions, on-chain governance).

## When to write a SIP

Write a SIP for any of:

- A consensus-rule change that requires a coordinated fork height (e.g. tokenomics-v2 fork, EVM gas-table change, signing-payload format).
- An economic-parameter change that affects the supply curve, halving cadence, fee split, or premine accounting.
- A wire-protocol change that bumps `SENTRIX_PROTOCOL` and requires a halt-all + simul-start cluster swap.
- A new precompile, host call, or system contract.
- A change to staking, slashing, jailing, or reward-distribution mechanics.
- A new public RPC surface (REST endpoint, JSON-RPC method, gRPC service, WebSocket subscription channel).

Skip a SIP for:

- Bug fixes that preserve protocol semantics.
- Internal refactors with no observable behaviour change.
- Tooling, CI, monitoring, or operator-facing infra.
- Documentation-only changes.

## Process

1. Read [SIP-1](sips/sip-1.md) — it specifies this process.
2. Copy [TEMPLATE.md](TEMPLATE.md) to `sips/sip-<next-number>.md` (next available unused integer).
3. Open a pull request against this repo. Status starts at `Draft`.
4. Discussion happens in PR review. Substantial changes update the SIP file in-place.
5. When the change is implemented and live on at least the testnet, the SIP is merged with status `Final`.
6. If a SIP is replaced or supersedes another, update both files' headers.

## Status values

| Status | Meaning |
|---|---|
| `Draft` | Open for discussion. Subject to substantive change. |
| `Review` | Author believes the proposal is complete; community review in progress. |
| `Last Call` | Final ~14-day comment window before acceptance. |
| `Accepted` | Implementation may begin. No mainnet ship yet. |
| `Final` | Live on mainnet. Update CHANGELOG entry in the chain repo to reference this SIP. |
| `Withdrawn` | Author abandoned the proposal. Kept for historical reference. |
| `Superseded` | Replaced by a later SIP. Header points at the replacement. |

## Index

| SIP | Title | Status | Type |
|---|---|---|---|
| [SIP-1](sips/sip-1.md) | SIP Process | Final | Process |

(More SIPs will land here as the process is exercised.)

## Reference orgs

This repository follows the proposal-repo pattern used by every major L1 ecosystem:

- [ethereum/EIPs](https://github.com/ethereum/EIPs) (Ethereum)
- [near/NEPs](https://github.com/near/NEPs) (NEAR Protocol)
- [aptos-foundation/AIPs](https://github.com/aptos-foundation/AIPs) (Aptos)
- [bnb-chain/BEPs](https://github.com/bnb-chain/BEPs) (BNB Chain)
- [sui-foundation/sips](https://github.com/sui-foundation/sips) (Sui)
- [avalanche-foundation/ACPs](https://github.com/avalanche-foundation/ACPs) (Avalanche)

## License

[CC0-1.0](LICENSE) — SIP text is in the public domain so anyone can quote, fork, or implement without restriction.
