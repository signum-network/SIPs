---
sip: 62
title: Strong random numbers for smart contracts
description: Non-blocking random numbers from a block that did not exist when the input was committed
author: ohager
discussions-to: https://github.com/signum-network/signum-node/issues/1077
status: Draft
type: Standard
category: Core
created: 2026-11-04
---

## Abstract

This SIP adds two AT API functions that return a random number derived from the generation signature of block s + K, where s is the height at which the input was committed and K = 4. When that block is not yet part of the chain, the functions return -1 and the AT keeps running. The value depends only on s and the inputs, never on the block in which the AT runs.

`GET_RANDOM_FOR_TX_IN_A` (0x0311) takes s from a transaction. `GET_RANDOM_AT_HEIGHT` (0x0312) takes s and a salt from the contract, for draws that no single transaction triggers. The existing `GET_RANDOM_ID_FOR_TX_IN_A` (0x0308) stays unchanged.

## Motivation

Games, lotteries and fair selections need random numbers that nobody knows when the bets are placed. Neither of the two sources available today provides this in a usable way.

### getWeakRandomNumber is predictable

SmartC's `getWeakRandomNumber()` reads the generation signature of the previous block (`PUT_LAST_BLOCK_GSIG_IN_A`, 0x040e). The generation signature is computed in advance:

```
gensig(H) = SHA256(gensig(H-1), generatorId(H-1))
```

So `gensig(H-1)` is fixed as soon as block H-2 is published, and anyone can compute it. A user who sees block H-2 knows the number that an AT running in block H will draw. The user sends a transaction only if the outcome is favourable; if it lands in block H-1, the result is certain. Nobody has to manipulate anything.

### GET_RANDOM_ID_FOR_TX_IN_A blocks the AT

`GET_RANDOM_ID_FOR_TX_IN_A` (0x0308) returns `SHA256(gensig(H-1), txId, senderPublicKey)` and is only available 15 blocks after the transaction. This makes it unpredictable for users, but:

- **It blocks.** Called too early, it puts the AT to sleep and repeats the call later. During that time, all other transactions wait. A contract with steady traffic stalls for up to 15 blocks per draw.
- **It depends on the run height.** H is the block in which the AT runs. If the AT runs later than 15 blocks after the transaction, it draws a different number. A block generator who leaves the AT out of a block re-rolls the result.
- **It needs a transaction.** A draw at the end of a round, not tied to one transaction, is not possible.

The functions of this SIP keep the unpredictability of 0x0308 and remove the three drawbacks.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

### Definitions

- **H** is the height of the block in which the AT runs.
- **K** is the random delay, a protocol constant: **K = 4**.
- **s** is the commit height: the height at which the input that the draw must not be known to was fixed.
- Block s + K is **available** if s + K ≤ H − 1.
- The **random value** R(s, salt) is computed as follows:
    1. Concatenate the 32-byte generation signature of block s + K, the 8-byte ID of the executing AT and the 8-byte salt. Both IDs are little-endian.
    2. Compute SHA-256 over these 48 bytes.
    3. Read the first 8 bytes of the hash as a little-endian 64-bit integer and clear its highest bit.
- R lies in the range 0 to 2^63 − 1. Negative return values are status codes.

### Function GET_RANDOM_FOR_TX_IN_A

| Property | Value |
| --- | --- |
| Name | `GET_RANDOM_FOR_TX_IN_A` |
| Function number | 0x0311 |
| Opcode | `EXT_FUN_RET` (0x35) |
| Assembly | `FUN @addr 0x0311` |
| Input | A1 = transaction ID |
| Output | `@addr` = R(s, txId) with s = height of the transaction; -1 if block s + K is not available; -2 if A1 is not a transaction in a block before H |
| Registers | A and B unchanged |

The result equals `GET_RANDOM_AT_HEIGHT` with B1 = height of the transaction and B2 = transaction ID. The transaction does not need to be addressed to the executing AT.

### Function GET_RANDOM_AT_HEIGHT

| Property | Value |
| --- | --- |
| Name | `GET_RANDOM_AT_HEIGHT` |
| Function number | 0x0312 |
| Opcode | `EXT_FUN_RET` (0x35) |
| Assembly | `FUN @addr 0x0312` |
| Input | B1 = commit height s; B2 = salt |
| Output | `@addr` = R(s, salt); -1 if block s + K is not available; -2 if s < 0 |
| Registers | A and B unchanged |

The contract is responsible for s. It MUST pass a height at or after the last input that must not know the result, for example the height at which a round closed. An older s yields a value that may already be known.

### Rules for both functions

1. The result MUST depend only on s, the salt and the AT ID. The same call MUST return the same value in every block after block s + K becomes available.
2. A call that returns -1 or -2 MUST NOT raise an error, stop the AT, put it to sleep or change any other state. The AT keeps running.
3. The comparison with H MUST NOT overflow for any 64-bit input.

### Cost

Both functions cost the standard `API_STEP_MULTIPLIER` (10) steps, like `GET_RANDOM_ID_FOR_TX_IN_A`.

### Activation

A new flux value at the next hard fork enables the functions. They MUST be available only to ATs whose creation height is at or after the activation height.

For all other ATs, a call MUST behave as today: an unknown `EXT_FUN_RET` function sets `@addr` to 0 and changes nothing. `GET_RANDOM_ID_FOR_TX_IN_A` (0x0308) MUST remain unchanged for all ATs.

### Node API

No changes.

## Rationale

- **Why the number comes from a future block.** Every node re-executes each AT and must reach the same state. True randomness such as Java's `SecureRandom` would differ per node and break consensus. A random value chosen by the block generator could not be verified and would let the generator pick the outcome. The only option is chain data that nobody knows when the input is committed.
- **Generation signature, not block signature or block hash.** `gensig(s+K)` depends on the generators of blocks s to s + K − 1, so users cannot predict it. Nobody can grind it; a generator can only withhold a block. A block signature or hash depends on the transaction selection and the timestamp, so the generator could recompute it until the outcome suits them.
- **K = 4.** At signing time, a user must guess the generators of four consecutive blocks to predict the value. The wait is about 16 minutes plus one block. A larger K makes prediction harder and draws slower; 4 balances both. K is fixed, because changing it would change pending draws.
- **Deterministic in s.** The value does not depend on the block in which the AT runs. Leaving the AT out of a block or running it late does not change the result. A draw can be resolved at any later time: by an alarm ([SIP-61](./sip-61.md)), by the next transaction, or by a resolve message anyone may send. Generation signatures are stored for all blocks, so there is no deadline for resolving.
- **-1 instead of sleeping.** A blocking call stalls all other transactions of the AT. With -1 the contract decides how to wait.
- **Two functions.** The transaction variant takes s from the chain, so the contract cannot pass a wrong height. The height variant covers draws without a triggering transaction, such as the end of a round, and multiple draws per height through the salt.
- **AT ID in the hash.** Different contracts drawing for the same height get independent values.
- **63 bits.** A non-negative result avoids the negative-modulo pitfall that the SmartC documentation warns about for `getWeakRandomNumber`, and leaves negative values for status codes.
- **New functions instead of changing 0x0308.** Deployed contracts rely on 0x0308 putting them to sleep. Returning -1 would make them treat -1 as a random number. Changing it only for new ATs would give one function two meanings depending on the creation height and confuse tools and audits.
- **Only new ATs.** Same rule as [SIP-58](./sip-58.md) to [SIP-61](./sip-61.md).
- **Function numbers.** 0x0311 and 0x0312 are the next free numbers in the 0x03 group (block and transaction information), after `GET_ALARM` (0x0310, [SIP-61](./sip-61.md)).

## Backwards Compatibility

Existing ATs are not affected. A pre-activation AT that calls 0x0311 or 0x0312 gets 0 today and keeps getting 0. `GET_RANDOM_ID_FOR_TX_IN_A` (0x0308) and `PUT_LAST_BLOCK_GSIG_IN_A` (0x040e) are unchanged. The node API is unchanged.

### Tooling

- SmartC needs built-ins, for example `getStrongRandom(txId)` and `getStrongRandomAt(height, salt)`.
- The SmartC documentation of `getWeakRandomNumber()` SHOULD state that its value is predictable by anyone one block in advance, not only open to tampering in theory.
- The AT API documentation SHOULD recommend the new functions instead of 0x0308 for new contracts.

Tooling is out of scope for this SIP.

## Security Considerations

The functions make the result unpredictable for users and ungrindable for block generators. They do not make it unbiasable: a generator can still withhold a block. Contracts whose stakes exceed a block reward SHOULD add a commit-reveal scheme between the participants.

| Threat | Mitigation |
| --- | --- |
| **Prediction by the user.** The user computes the result before sending. | The result depends on the generators of blocks s to s + K − 1, which nobody knows at signing time. |
| **Grinding by the generator.** The generator tries transaction sets or timestamps until the outcome suits them. | The generation signature does not depend on block content, so there is nothing to vary. |
| **Withholding.** The generator of block s + K − 1 knows `gensig(s+K)` before publishing. If the outcome is bad for them, they discard the block, and another generator produces block s + K − 1 with a different value. | Residual risk. It costs the block reward and fees, gives one re-roll to a random new value, and only works if the generator wins that block. Commit-reveal for high stakes. |
| **Re-roll by delaying the AT.** A generator leaves the AT out of a block to change the result. | The result depends only on s, not on the block in which the AT runs. |
| **Wrong commit height.** The contract passes an old s to `GET_RANDOM_AT_HEIGHT`, so the value is already known. | Contract logic. `GET_RANDOM_FOR_TX_IN_A` takes s from the chain and SHOULD be preferred where a transaction triggers the draw. |
| **Late inputs.** A user sends a bet after block s + K is known, and the contract still accepts it for that draw. | The contract MUST reject inputs for a draw after its commit height s. |
| **Correlated draws.** Several contracts draw for the same height. | The AT ID and the salt are part of the hash, so the values are independent. |
| **Rollback.** A reorganisation replaces block s + K. | The draw and its effects roll back together; the new chain gives a new value. Payouts based on a draw SHOULD wait for confirmations off chain. |

## Test Cases

| Case | Expected |
| --- | --- |
| `GET_RANDOM_AT_HEIGHT` with s = H − K − 1 | R(s, salt), a value ≥ 0 |
| `GET_RANDOM_AT_HEIGHT` with s = H − K | -1; AT keeps running |
| `GET_RANDOM_AT_HEIGHT` with s far in the future, including 2^63 − 1 | -1; no overflow |
| `GET_RANDOM_AT_HEIGHT` with s < 0 | -2 |
| Same s and salt in two different later blocks | Same value |
| Same s, two different salts | Different values |
| Same s and salt, two different ATs | Different values |
| `GET_RANDOM_FOR_TX_IN_A` for a transaction at height s, called at H ≥ s + K + 1 | Equals `GET_RANDOM_AT_HEIGHT` with B1 = s, B2 = transaction ID |
| `GET_RANDOM_FOR_TX_IN_A` called at H ≤ s + K | -1; AT keeps running |
| `GET_RANDOM_FOR_TX_IN_A` with an unknown transaction ID | -2 |
| Reference vector: fixed generation signature, AT ID and salt | Value from the reference implementation, to be added with the implementation |
| AT created before activation calls 0x0311 or 0x0312 | Returns 0; nothing changes |
| AT created after activation calls 0x0308 | Behaves as today |

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
