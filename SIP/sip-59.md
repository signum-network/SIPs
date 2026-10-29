---
sip: 59
title: Get own AT ID
description: Smart contracts can read their own AT ID through a new AT API function
author: ohager
discussions-to: https://github.com/signum-network/signum-node/issues/1072
status: Draft
type: Standard
category: Core
created: 2026-10-29
---

## Abstract

This SIP adds one AT API function, `GET_AT_ID` (0x030e), that returns the executing AT's own ID. Today an AT can learn its creator, its code hash and its activation fee, but not its own ID. The function is a read-only getter with standard step cost. It is available only to ATs created at or after the activation height.

## Motivation

No AT API function writes the AT's own ID into a register or memory. The node uses the ID internally as a default (B2 == 0 for `GET_CODE_HASH_ID` and `GET_ACTIVATION_FEE`, A3 == 0 for `GET_MAP_VALUE_KEYS_IN_A`, the map owner in `SET_MAP_VALUE_KEYS_IN_A`), but never hands it to the contract.

Use cases that need the ID as a value:

- **Reading its own entry in another contract's map.** A registry or factory AT that keys entries by contract ID can only be queried with `GET_MAP_VALUE_KEYS_IN_A` if the caller knows its own ID as key1 or key2.
- **Domain separation for signatures and hash commitments.** A message checked with `CHECK_SIG_B_WITH_A` or a hash checked with `CHECK_SHA256_A_WITH_B` should include the contract's ID. Otherwise a signature or preimage made for one instance can be replayed against another instance of the same code.
- **Self-describing messages.** Payloads sent off-chain or to other chains (for example in atomic swap flows) can carry the contract's ID without trusting a value supplied at deployment.

Current workarounds do not work:

- **Creator sends the ID after deployment.** The AT cannot verify the value, so the contract must trust its creator.
- **Matching a candidate ID against own properties.** Code hash, creator and activation fee are identical for every instance deployed from the same code by the same creator, so a match proves nothing.
- **Map nonce round-trip.** Writing a nonce to the own map and reading it back under a candidate ID costs at least two rounds. The nonce is public after the first block, so another AT can write the same entry.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

### Function

| Property | Value |
| --- | --- |
| Name | `GET_AT_ID` |
| Function number | 0x030e |
| Opcode | `EXT_FUN_RET` (0x35) |
| Assembly | `FUN @addr 0x030e` |
| Inputs | none |
| Output | `@addr` = the executing AT's ID as a signed 64-bit long |
| Registers | A and B unchanged |

The AT ID is the ID of the AT's creation transaction, the same value the node stores as the AT's account ID. The function reads it from the machine state; it needs no database access.

### Cost

Standard API cost: `API_STEP_MULTIPLIER` (10) steps, the same as `GET_CODE_HASH_ID` or `GET_ACTIVATION_FEE`. No special case in `getNumSteps` is needed.

### Activation

The function is enabled by a new flux value at the next hard fork. It MUST be available only to ATs whose creation height is at or after the activation height.

For all other ATs the call MUST behave as today: an unknown `EXT_FUN_RET` function sets `@addr` to 0. A real AT ID is never 0, so contracts can treat 0 as "not available".

## Rationale

- **`EXT_FUN_RET` instead of writing B.** `GET_CODE_HASH_ID` and `GET_ACTIVATION_FEE` return single values the same way. Writing B1 like `B_TO_ADDRESS_OF_CREATOR` would clobber B, which often holds a recipient or hash.
- **0x030e.** It is the next free number in the 0x03 group (block, transaction and AT info), right after `B_TO_ASSET_IDS_OF_TX_IN_A` (0x030d).
- **Separate SIP instead of bundling with SIP-58.** The getter is independent and uncontroversial.
- **Only new ATs.** Same rule as SIP-58: no existing contract changes behaviour, so no scan of deployed ATs is needed.
- **No creation-transaction getter.** The AT ID already is the creation transaction's ID.

## Backwards Compatibility

Existing ATs are not affected. A pre-activation AT that calls 0x030e already gets 0 today, and keeps getting 0. No other AT API function changes. Compiler support (SmartC) and the AT API documentation need a new entry; tooling is out of scope for this SIP.

## Test Cases

| Case | Expected |
| --- | --- |
| AT created at or after activation calls `GET_AT_ID` | `@addr` = AT ID (= creation transaction ID) |
| Two instances of the same code by the same creator | Each gets its own, different ID |
| AT created before activation calls 0x030e after activation | `@addr` = 0 |
| Any AT calls 0x030e before activation | `@addr` = 0 |
| Registers after the call | A1..A4 and B1..B4 unchanged |
| Step cost | 10 steps |
| Round trip: `GET_AT_ID` as key3 of `GET_MAP_VALUE_KEYS_IN_A` | Same value as with A3 = 0 |

## Reference Implementation

| File | Change |
| --- | --- |
| `src/brs/at/OpCode.java` | Add `GET_AT_ID = 0x030e` |
| `src/brs/at/AtApi.java` | Add `long getAtId(AtMachineState state)` |
| `src/brs/at/AtApiImpl.java` | Delegate to the platform, like `getCodeHashId` |
| `src/brs/at/AtApiPlatformImpl.java` | Return `AtApiHelper.getLong(state.getId())` |
| `src/brs/at/AtApiController.java` | Add the case to `func`, gated by the flux value at the AT's creation height |
| `src/brs/fluxcapacitor/FluxValues.java` | New flux value |
| `test/java/brs/at/` | Tests for the cases above |

## Security Considerations

The AT ID is public on chain, so exposing it to the contract reveals nothing new. The function reads no other account's data and has a fixed cost, so it adds no denial-of-service vector.

Contracts that use the ID for domain separation MUST NOT accept 0 as a valid ID. A contract created before activation gets 0, and a signature or commitment bound to 0 would again be valid for every such instance.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
