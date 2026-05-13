---
sip: <next available integer — leave blank in PR; maintainer assigns at merge>
title: <short descriptive title, lowercase, no trailing period>
author: <Your Name <your@email>>, ...
status: Draft
type: <Standards Track | Process | Informational>
category: <if Standards Track: Core | Networking | Interface | ERC-equivalent>
created: <YYYY-MM-DD>
requires: <SIP numbers, if any>
replaces: <SIP numbers, if any>
superseded-by: <SIP number, if applicable; usually filled by maintainer>
---

## Abstract

One paragraph (≤ 200 words) summarising what this SIP changes and why. A reader should be able to decide from this section alone whether the rest is relevant to them.

## Motivation

Why is the current behaviour insufficient? What concrete problem does this SIP solve? Reference incident reports, audit findings, or ecosystem-wide friction the change addresses. Avoid abstract aspirations — point at the thing that hurts today.

## Specification

The normative section. Be precise. Anyone implementing a Sentrix-compatible client should be able to write conforming code from this section without consulting the chain repo.

For consensus changes, include:

- Affected modules + functions (`crates/sentrix-bft`, `crates/sentrix-staking`, etc.)
- Wire-format diffs (proto schemas, byte layouts, hash inputs)
- State-machine transitions, including edge cases
- Activation height (`<NAME>_HEIGHT` env var) and forking semantics
- Determinism invariants the change must preserve

For interface changes, include:

- Endpoint URL + method + request/response schema
- Backwards-compat plan (deprecate old → ship new → remove old)

## Rationale

Why this design over the alternatives considered? Anchor the trade-offs in concrete numbers when possible (block-time impact, gas cost, state size, network bandwidth, attack surface).

## Backwards Compatibility

What breaks. What stays. Migration path for affected node operators, validators, dApp developers, and indexer consumers.

If the change requires a coordinated fork, name the fork height variable and the testnet bake plan.

## Security Considerations

The class of failures this change introduces or removes. Threat model. New attack surfaces. Slashing-condition impact. Recovery procedures if the fork goes wrong.

## Reference Implementation

Link to the PR on `sentrix-labs/sentrix` (or related repo) implementing the change. Required before status moves from `Accepted` → `Final`.

## Copyright

This SIP is dedicated to the public domain under CC0-1.0.
