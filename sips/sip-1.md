---
sip: 1
title: SIP Process
author: Satya Kwok <satya@sentrixchain.com>
status: Final
type: Process
created: 2026-05-13
---

## Abstract

SIP-1 defines the process by which all subsequent Sentrix Improvement Proposals are written, reviewed, accepted, and shipped. It is intentionally lightweight: a SIP is a markdown file with a small frontmatter block, opened as a pull request against this repository, discussed in PR review, and merged when the proposal is implemented and live.

## Motivation

Sentrix Chain is at the size where most changes still ship through normal pull requests against the chain repo. A separate SIP repo exists because three classes of change benefit from a slower, more public review surface than a chain-repo PR comment thread provides:

1. **Consensus rule changes** — anything that requires a coordinated fork height. Implementers and validator operators need a stable URL they can cite when reasoning about why their node has to upgrade by a specific block height.
2. **Economic parameter changes** — supply curve, halving cadence, fee split, premine accounting. Token holders have a stake in these decisions that goes beyond what reviewers on a single PR can represent.
3. **Wire-protocol changes** — anything that bumps `SENTRIX_PROTOCOL` and requires a halt-all + simul-start cluster swap. External tooling (block explorers, indexers, wallets) needs a stable spec to track.

A SIP is *not* a vote. The protocol does not have on-chain governance over these decisions. The SIP process is a documentation and review surface — final acceptance still flows through the maintainers of the chain repo and the validators who operate the network.

## Specification

### File layout

Each SIP lives at `sips/sip-<integer>.md`. The integer is monotonically increasing and assigned at merge time by a maintainer; PR authors leave it blank in the frontmatter.

### Frontmatter fields

| Field | Required | Notes |
|---|---|---|
| `sip` | yes (assigned at merge) | Integer, no leading zeros. |
| `title` | yes | Short, lowercase, no trailing period. |
| `author` | yes | `Name <email>`, comma-separated for multiple. |
| `status` | yes | One of `Draft`, `Review`, `Last Call`, `Accepted`, `Final`, `Withdrawn`, `Superseded`. |
| `type` | yes | `Standards Track`, `Process`, or `Informational`. |
| `category` | if `type=Standards Track` | `Core`, `Networking`, `Interface`, or `ERC-equivalent`. |
| `created` | yes | ISO-8601 date. |
| `requires` | optional | SIPs this depends on. |
| `replaces` | optional | SIPs this replaces. |
| `superseded-by` | optional | Filled by maintainer when superseded. |

### Status transitions

```
Draft → Review → Last Call → Accepted → Final
                                ↘
                              Superseded (by another SIP)
                                ↘
                              Withdrawn (by author)
```

- `Draft → Review`: author asserts the SIP is complete enough for serious review.
- `Review → Last Call`: maintainer determines the proposal is well-formed and opens a 14-day final-comment window.
- `Last Call → Accepted`: implementation may begin. No mainnet ship yet.
- `Accepted → Final`: change is live on at least one production network (mainnet or long-running testnet) and the chain repo `CHANGELOG.md` references this SIP number.

### Required sections

Every SIP, regardless of type, includes:

- Abstract (≤ 200 words)
- Motivation
- Specification
- Rationale
- Backwards Compatibility
- Security Considerations
- Reference Implementation (filled before `Final`)
- Copyright

The `TEMPLATE.md` at the repo root encodes these sections.

### Numbering

SIP numbers are dense — every integer from 1 onward is used in order. There are no reserved ranges (no separate "Core SIP" 100-series). Process SIPs, Standards Track SIPs, and Informational SIPs share the same sequence.

### Withdrawal and supersession

An author may withdraw a `Draft` or `Review` SIP at any time by changing the status. The file stays in the repo for historical reference; the PR is closed, not reverted.

A SIP that becomes `Final` cannot be edited normatively. If its design needs revision, a new SIP is opened that supersedes it; both files' headers are updated to point at each other.

## Rationale

This process tracks what every major L1 ecosystem already does (Ethereum EIPs, NEAR NEPs, Aptos AIPs, BNB BEPs, Sui SIPs, Avalanche ACPs). The choices specific to Sentrix:

- **Markdown in a separate repo** rather than a wiki or design doc tool — proposals get the same review surface as code, with version history and offline access.
- **Single integer sequence** rather than separate ranges per type — keeps citations simple.
- **No on-chain governance** — the protocol does not yet have a token-weighted vote mechanism, and adding one to gate SIP acceptance would be a multi-quarter project. The SIP process is a documentation and review surface; final ship authority stays with the chain maintainers.

## Backwards Compatibility

Not applicable. SIP-1 establishes the process; there is nothing to break.

## Security Considerations

Not applicable to a process SIP.

## Reference Implementation

This repository.

## Copyright

This SIP is dedicated to the public domain under CC0-1.0.
