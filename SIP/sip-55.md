---
sip: 55
title: Raise smart contract size limits
description: Raise the maximum code, data and user stack pages of smart contracts
author: ohager
discussions-to: https://github.com/signum-network/signum-node/issues/1061
status: Draft
type: Standard
category: Core
created: 2026-10-21
requires: 37
---

## Abstract

This SIP raises the limits for creating smart contracts (ATs): code pages from 40 to **127** (32,512 bytes), data pages from 10 to **32** (8 KB) and user stack pages from 10 to **16**. The call stack stays at **10** pages. The creation bytes format does not change, and existing contracts are not affected. The limits activate at a new fork height.

## Motivation

[SIP-37](./sip-37.md) raised the code limit to 40 pages (10 KB). More complex contracts, such as DeFi, games and multi-role contracts, regularly hit this limit and have to be split into several contracts. That raises fees and complexity.

Larger code alone does not help much: the 10 data pages (320 variables) and the 10 user stack pages, which SmartC uses for function arguments, also become a bottleneck.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

### New limits

From the fork height, AT creation transactions MUST be validated against the following limits (page size 256 bytes):

| | current | new |
|---|---|---|
| Code pages | 40 | **127** (32,512 bytes) |
| Data pages | 10 | **32** (8,192 bytes) |
| User stack pages | 10 | **16** (4,096 bytes) |
| Call stack pages | 10 | **10** (unchanged) |

### Activation

- The limits MUST be determined by the **block height** of the creation transaction, not by the AT version written in the creation bytes. Contracts therefore need no new AT version.
- Before the fork height, creation transactions that exceed the current limits MUST still be rejected.
- The limits apply only at creation. Existing contracts MUST NOT be affected.

### Unchanged

- Creation bytes format, including the length encodings.
- Fee per page (0.1 SIGNA) and size-based fees.
- Execution fees and the maximum steps per execution.

## Rationale

### 127 instead of 128 code pages

The code and data lengths in the creation bytes are encoded as a 2-byte `short` below 32,768 bytes and as a 4-byte `int` from 32,768 bytes on. 128 pages (exactly 32,768 bytes) would use the 4-byte encoding on mainnet for the first time. The node can parse it, but it has never been used and the tooling (SmartC, signumjs) is unverified. With 127 pages, contracts lose 256 bytes and no new encoding path enters consensus. The data limit of 32 pages also stays far below this threshold.

### Data and stacks

- **32 data pages** (1024 variables) gives room for larger contracts but keeps data pages as working memory. Maps ([SIP-38](./sip-38.md)) remain the right tool for large data sets.
- **16 user stack pages**: SmartC passes function arguments on the user stack, so recursion and deep call chains run out of space with 10 pages.
- **Call stack unchanged:** 10 pages allow 320 nested calls, which is enough in practice.

### Height-based limits

Height-based limits avoid a new AT version that every compiler and SDK would have to emit. The tooling only needs to raise its own limits.

## Backwards Compatibility

This is a hard-forking change and breaks compatibility with old fully-validating nodes. It MUST NOT be deployed without widespread consensus.

- Existing contracts are not affected.
- SmartC, signumjs and the SmartC testbed must raise their limits to make the new sizes usable.

## Test Cases

For each limit, before and after the fork height:

- code pages 127 accepted, 128 rejected
- data pages 32 accepted, 33 rejected
- user stack pages 16 accepted, 17 rejected
- call stack pages 10 accepted, 11 rejected
- before the fork, a contract with 41 code pages is rejected

## Reference Implementation

Tracked in [signum-node#1061](https://github.com/signum-network/signum-node/issues/1061). Affected components: `AtConstants`, `AtController.checkCreationBytes`, `FluxValues`.

The current node looks up the data and stack limits with the AT version passed as a block height (`atVersion(version)`), which always resolves to AT version 1 on mainnet. Nobody notices today because all versions use 10. This MUST be fixed together with this SIP, otherwise the new data and stack limits are silently ignored. The fix is consensus-neutral for existing versions.

## Security Considerations

### Creation size and fees

- Max pages per AT: 127 + 32 + 16 + 10 = 185, so the page fee rises from 7 to 18.5 SIGNA.
- Max creation transaction: about 41 KB including init data. That is about 11% of the block payload (about 375 KB), plus about 2.3 SIGNA size fee.
- At these fees, filling blocks with large creation transactions is expensive.

### State storage

The main cost is not the code but the **state**. The code is stored once per contract. On every execution, the node stores a new state row containing the **full** compressed state (data + stacks), and keeps these rows for the rollback window of 1440 blocks.

- Max raw state grows from 7.5 KB to 14.8 KB.
- Page fees are paid once at creation, while each execution only pays steps. A contract that fills its state once with incompressible data and then runs every block with a few steps occupies about 21 MB in the rollback window, compared to about 11 MB today.

This asymmetry already exists and is roughly doubled by this SIP. Before activation, the state storage on mainnet and the block processing time with several large contracts SHOULD be measured on testnet. Node implementations MAY reduce the cost later, for example by storing only state changes. That would not need a consensus change.

### Encoding

Keeping the lengths below 32,768 bytes avoids activating an untested parsing path in consensus code.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
