---
sip: 53
title: Controllable transaction bundling for ATs
description: Let smart contracts control how outgoing transactions are bundled and fix silent message loss on overflow
author: ohager
discussions-to: https://github.com/signum-network/signum-node/issues/1059
status: Draft
type: Standard
category: Core
created: 2026-10-21
---

## Abstract

This SIP adds a new AT API function `SET_TX_BUNDLE` (`0x0411`). With it, a smart contract controls how the transactions it creates during one execution are bundled. Today all sends to the same recipient and asset are merged into a single transaction. It also fixes a bug where messages to the same recipient are silently discarded once the accumulated message exceeds the maximum message length. Both changes activate at a new fork height.

## Motivation

Within one execution, the node collects all transactions an AT creates and keys them by **recipient + assetId**, or by transaction type if there is no recipient. A second send with the same key does not create a new transaction:

- amounts and quantities are **summed**: sending 3 SIGNA to A and then 2 SIGNA to A results in one transaction of 5 SIGNA
- the transaction type is not part of the key, so a payment and a message to the same recipient are merged, and the resulting transaction takes the type of the last action
- messages are **concatenated** (AT version ≥ 3) while the existing message is shorter than `MAX_ARBITRARY_MESSAGE_LENGTH - 32` (968 bytes)

As a result, a contract cannot send two separate transactions or messages to the same recipient in one block, and cannot run independent message streams to the same recipient.

**Silent message loss:** `sendAToAddressInB` always sends 32-byte messages. After 31 messages to the same recipient (992 bytes), the next one discards the 992 accumulated bytes and only the latest 32 bytes remain. The contract cannot detect this.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

### New operation

- `SET_TX_BUNDLE = 0x0411` (`EXT_FUN_DAT`): selects the bundle that subsequent sends are assigned to.

The bundling key becomes **(bundle, recipient, assetId)**.

| Argument `n` | Behavior |
|---|---|
| `0 ≤ n ≤ Integer.MAX_VALUE` | select bundle `n` |
| `n < 0` | select the next free bundle (highest used bundle + 1). If the highest used bundle is already `Integer.MAX_VALUE`, this is a no-op |
| `n > Integer.MAX_VALUE` | no-op |

- Bundle `0` is the default and MUST behave exactly as today.
- The selection MUST reset to `0` at the start of every execution.
- The function costs the standard `apiStepMultiplier` (10 steps), like every API function.
- Switching back to an earlier bundle merges into the existing transaction of that bundle, which keeps its original position in the output order.

### Unique transaction IDs

AT transaction IDs are derived only from the transaction bytes. Without further changes, `send(A, 1)`, `SET_TX_BUNDLE(-1)`, `send(A, 1)` would produce two byte-identical transactions with the same ID.

To prevent this, the node MUST write the bundle number into the `ecBlockHeight` field of the AT transaction (currently always `0`). Bundle `0` therefore stays byte-identical to today's transactions.

### Message overflow

After the fork height, the following applies to **all** ATs, regardless of AT version:

- When appending a message would exceed the maximum message length, the current transaction MUST be closed and keep its position.
- The stream `(bundle, recipient, assetId)` MUST continue in a **new transaction** appended at the end of the output order.
- Other streams and the current bundle selection MUST NOT be affected.
- Overflow transactions MUST use negative `ecBlockHeight` values (−1, −2, … per execution), so they never collide with a bundle selected via `SET_TX_BUNDLE`.

### Activation

- All rules in this SIP activate at a new fork height.
- Before the fork height, `0x0411` MUST remain a no-op (unknown functions return `0`). Activation is by block height, not by AT version, so a contract that calls it before the fork behaves the same before and after.

## Rationale

- **Explicit bundles instead of "never merge":** merging is the current behavior and many contracts rely on it, so bundle `0` keeps it unchanged. Bundles also allow several independent streams to the same recipient.
- **`ecBlockHeight` for uniqueness:** it is unused for AT transactions (`EconomicClustering.verifyFork` only checks user transactions), it keeps bundle `0` byte-identical, and it needs no format change.
- **Continuing instead of raising an error on overflow:** an error would likely leave the contract stopped, which is much harder to debug than a transparent continuation.
- **No extra step cost:** AT transactions do not count against the block payload. Each executed AT costs a fixed `AT_ID_SIZE + 16` bytes. The effective bound is steps: `SET_TX_BUNDLE(-1)` + send costs about the same 20 steps as set-recipient + send, which is already possible today, so the worst-case number of transactions per execution does not increase.

## Backwards Compatibility

This is a hard-forking change and breaks compatibility with old fully-validating nodes. It MUST NOT be deployed without widespread consensus.

- Existing contracts that do not call `SET_TX_BUNDLE` keep their current bundling behavior.
- Contracts that send more than 968 bytes of messages to one recipient per execution will now produce additional transactions instead of losing data.
- Explorers, wallets and API clients will see non-zero `ecBlockHeight` values on AT transactions (≥ 0 for bundles, < 0 for overflow transactions).

## Test Cases

- Bundling key `(bundle, recipient, assetId)`: same key merges, different bundle does not.
- `SET_TX_BUNDLE(n)` with `n ≥ 0`, `n < 0`, `n > Integer.MAX_VALUE`, and `n < 0` when bundle `Integer.MAX_VALUE` is already used.
- Selection resets between executions.
- Switching back to an earlier bundle keeps that transaction's position.
- Message overflow creates a continuation transaction with a negative `ecBlockHeight` and loses no bytes.
- Transaction IDs stay unique within one block.
- Before the fork height `0x0411` is a no-op and the old overflow behavior applies.

## Reference Implementation

Tracked in [signum-node#1059](https://github.com/signum-network/signum-node/issues/1059). Affected components: `OpCode`, `AtApiController` / `AtApiPlatformImpl`, `AtMachineState` (bundling key, overflow handling), `AtTransaction` (`ecBlockHeight`), `FluxValues`.

Tooling: SmartC SHOULD expose `flush()` → `SET_TX_BUNDLE(-1)` and `flush(n)` → `SET_TX_BUNDLE(n)`. The SmartC testbed/simulator SHOULD emulate the new bundling and overflow behavior.

## Security Considerations

- **Duplicate transaction IDs** would break block persistence (duplicate primary key). The `ecBlockHeight` rule guarantees unique transaction bytes per AT and block.
- **Transaction spam:** the number of transactions per execution stays bounded by the step limit (max 1,000,000 steps). An extra transaction costs about 0.02 SIGNA in step fees, which is above the minimum fee of a regular transaction.
- **Fork safety:** activation by block height prevents contracts that already call `0x0411` from changing behavior retroactively.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
