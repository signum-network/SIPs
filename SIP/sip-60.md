---
sip: 60
title: Transferable smart contract ownership
description: Smart contracts get a transferable owner next to the immutable creator
author: ohager
discussions-to: https://github.com/signum-network/signum-node/issues/1074
status: Draft
type: Standard
category: Core
created: 2026-10-30
---

## Abstract

This SIP gives every smart contract (AT) an owner next to its immutable creator. At creation the owner is the creator. Only the AT itself can change its owner, through a new AT API function `SET_OWNER` (0x0413); no transaction can transfer an AT from outside. A second function, `GET_OWNER` (0x030f), reads the owner of the executing AT or of any other AT.

`SET_OWNER` is ignored when the new owner is not an existing account. It never fails and never stops the contract; it returns 1 when the owner was set and 0 when it was ignored. The node indexes owners and adds the API endpoint `getOwnerContracts`. The creator and `B_TO_ADDRESS_OF_CREATOR` stay unchanged.

## Motivation

Today a smart contract cannot change hands on protocol level. The creator is fixed at deployment, and many contracts use it as their admin through `B_TO_ADDRESS_OF_CREATOR` (0x030b).

Contracts that need transferable ownership keep the owner in their own memory, for example SRC-40 NFTs ([SIP-40](./sip-40.md)). This has three drawbacks:

- **No index.** A node cannot answer "which contracts does account X own?" without reading the memory of every AT. Wallets and explorers can only list contracts by creator (`getAccountATs`).
- **No common reading.** Each standard stores the owner at its own memory address. Other contracts and tools must know the layout of each standard to find it.
- **Repeated code.** Every contract implements its own owner variable and transfer logic, and each copy can have its own bugs.

Use cases that need a common, indexed owner:

- **Selling a contract.** A service contract, a token-issuing contract or a liquidity pool changes hands together with its admin rights.
- **NFTs.** A new NFT standard can use the protocol owner directly, so wallets list NFTs with `getOwnerContracts`.
- **Composition.** One contract (for example a DAO) owns other contracts and administers them by sending transactions.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

### Owner

- Every AT has exactly one owner: an account ID.
- When an AT is created, its owner MUST be its creator. The creation transaction has no field for a different owner.
- For ATs created before the activation height, the owner is their creator. These ATs cannot call `SET_OWNER`, so their owner never changes.
- The creator MUST NOT change. `B_TO_ADDRESS_OF_CREATOR` (0x030b) and the `creator` field of the node API keep their meaning.
- The owner can only be changed by the AT itself, through `SET_OWNER`. There is no transaction type that changes the owner of an AT.

### Function GET_OWNER

| Property | Value |
| --- | --- |
| Name | `GET_OWNER` |
| Function number | 0x030f |
| Opcode | `EXT_FUN_RET` (0x35) |
| Assembly | `FUN @addr 0x030f` |
| Input | B2 = AT ID to query; 0 = the executing AT |
| Output | `@addr` = the owner's account ID; 0 if B2 is not the ID of an AT |
| Registers | A and B unchanged |

The input follows `GET_CODE_HASH_ID` and `GET_ACTIVATION_FEE`. Querying an AT created before activation returns its creator.

### Function SET_OWNER

| Property | Value |
| --- | --- |
| Name | `SET_OWNER` |
| Function number | 0x0413 |
| Opcode | `EXT_FUN_RET` (0x35) |
| Assembly | `FUN @addr 0x0413` |
| Input | B1 = new owner's account ID |
| Output | `@addr` = 1 if the owner was set, 0 if the call was ignored |
| Registers | A and B unchanged |

Rules:

1. `SET_OWNER` MUST only change the owner of the executing AT. No input selects another AT.
2. The call MUST be ignored and return 0 if no account with ID B1 exists. Account ID 0 never exists, so the owner can never be 0.
3. An account exists if the account table has an entry for it at the end of the previous block. Accounts created in the current block do not count. All nodes therefore get the same result, whether they generate or validate the block.
4. Any existing account MAY become owner: a normal account, another AT, or the executing AT itself.
5. Setting the current owner again returns 1 and changes nothing.
6. An ignored call MUST NOT raise an error, stop the AT or change any other state. It costs the normal steps.
7. If an AT calls `SET_OWNER` several times in one run, the last successful call wins.

### Visibility within a block

An owner change is written when the block is applied, like map updates from `SET_MAP_VALUE_KEYS_IN_A`.

- The executing AT sees its own pending owner: `GET_OWNER` with B2 = 0 or its own ID returns the new value in the same run.
- Other ATs in the same block see the owner as of the end of the previous block. The result does not depend on the order in which ATs run.
- A rollback of the block MUST restore the previous owner.

### Cost

Both functions cost the standard `API_STEP_MULTIPLIER` (10) steps, like `GET_ACCOUNT_BALANCE`. `getNumSteps` needs no special case.

### Node API

- All responses that describe an AT (`getAT`, `getATs`, `getAccountATs`, `getATDetails`) MUST add the fields `owner` and `ownerRS`, next to `creator` and `creatorRS`.
- New endpoint `getOwnerContracts`: lists the ATs whose current owner is the given account, including ATs created before activation (owner = creator). Parameters as in `getAccountATs`: `account` (required), `machineCodeHashId` (optional), `firstIndex`, `lastIndex`. Response: `ats` array and `nextIndex`, newest first.
- `getAccountATs` keeps listing ATs by creator.

### Activation

A new flux value at the next hard fork enables the functions. `GET_OWNER` and `SET_OWNER` MUST be available only to ATs whose creation height is at or after the activation height.

For all other ATs, a call MUST behave as today: an unknown `EXT_FUN_RET` function sets `@addr` to 0 and changes nothing.

## Rationale

- **Only the AT changes its owner.** A protocol owner gives no control by itself; the contract's code decides what the owner may do. So the code also decides when ownership moves. A transfer transaction would bypass the contract, for example during a running auction or while funds are locked.
- **Trustless sale.** Because the contract controls the transfer, it can sell itself atomically: the buyer pays the contract, the contract sets the buyer as owner and forwards the payment to the old owner. No escrow is needed.
- **No confirmation by the new owner.** A two-step transfer would need a new transaction type for the acceptance. The existence check already prevents the main accident, a transfer to a mistyped ID that no account uses.
- **Ignore instead of fail.** A failing `SET_OWNER` would stop or freeze the contract, possibly with funds in it. Returning 0 lets the contract react, for example by refunding a payment.
- **Owner 0 not allowed.** A contract can give up ownership by owning itself, so 0 is not needed. In many contracts 0 means "not set", so an owner 0 would be ambiguous. Alias transfers to 0 are rejected too. A later SIP can still allow it if a use case appears.
- **Separate owner, unchanged creator.** Many deployed contracts use the creator as admin. Changing what `B_TO_ADDRESS_OF_CREATOR` returns would hand their admin rights to whoever holds the owner.
- **Only new ATs.** Same rule as [SIP-58](./sip-58.md) and [SIP-59](./sip-59.md): no deployed contract changes behaviour.
- **Function numbers.** 0x030f is the next free number in the 0x03 group (AT information), after `GET_AT_ID` (0x030e, [SIP-59](./sip-59.md)). 0x0413 is the next free number in the 0x04 group (state changes), after 0x0411 ([SIP-53](./sip-53.md)) and 0x0412 ([SIP-54](./sip-54.md)).
- **`EXT_FUN_RET` for both.** It returns a value without clobbering A or B, which often hold a recipient or hash.
- **Storage (non-normative).** A separate versioned table `at_owner (at_id, owner_id, height, latest)` is RECOMMENDED. The `at` table holds the compressed code, and versioning it would copy the code on every owner change. An index on `(owner_id, latest)` serves `getOwnerContracts`.

## Backwards Compatibility

Existing ATs are not affected. A pre-activation AT that calls 0x030f or 0x0413 gets 0 today and keeps getting 0. The creator, `B_TO_ADDRESS_OF_CREATOR` and `getAccountATs` are unchanged. API responses gain two fields; clients that ignore unknown fields need no change.

### Existing contracts in explorers and wallets

For ATs created before activation, the node reports owner = creator. Some of these contracts manage ownership in their own memory, most notably SRC-40 NFTs ([SIP-40](./sip-40.md)). For them, the reported owner can differ from the real one.

Explorers and wallets SHOULD show the reported owner with a hint when they recognize such a contract, for example by its code hash:

> Ownership is determined inside the contract logic. SRC-40 was created before SIP-60.

New standards SHOULD use the protocol owner only, so that the reported owner and the real one cannot differ.

### Tooling

SmartC needs built-ins for both functions, for example `getOwner()`, `getOwnerOf(atId)` and `setOwner(account)`. SignumJS and the AT API documentation need the new fields and the new endpoint. Tooling is out of scope for this SIP.

## Security Considerations

The protocol owner records who owns a contract; it does not grant control. Control comes only from contract code that checks `GET_OWNER`. Buyers MUST verify the code before they pay for a contract.

| Threat | Mitigation |
| --- | --- |
| **Owner without control.** A contract checks the creator or its own variable instead of `GET_OWNER`. The buyer pays and is listed as owner, but the seller keeps control. | Only ATs created after activation can change their owner. Explorers show creator and owner separately. Buyers rely on audited templates, verified by code hash. |
| **Takeover of another AT.** One AT changes the owner of another. | `SET_OWNER` has no AT input; it only acts on the executing AT. `GET_OWNER` reads other ATs but cannot write them. |
| **Takeover by external transaction.** | No transaction type changes the owner. Every change runs through the contract's own code. |
| **Missing sender check in the contract.** Anyone who sends the right message becomes owner. | Not preventable by the protocol. SmartC SHOULD ship a reviewed transfer template; contracts SHOULD be audited. |
| **Transfer to a mistyped ID.** | Ignored if no account with that ID exists. A mistyped ID that hits an existing account cannot be detected; contracts MAY require a confirmation message from the new owner. |
| **Contract stopped by a bad input.** | An ignored call returns 0 and never raises an error or stops the AT. |
| **Stale owner in delayed processing.** ATs process transactions after a delay; `GET_OWNER` returns the current owner, not the owner when the transaction was sent. | Commands of a previous owner are rejected after a transfer, as intended. Contracts that must also reject commands sent before a transfer compare the transaction height with the height of their last owner change. |
| **Order of ATs in a block.** | Other ATs see the owner as of the end of the previous block, independent of execution order. |
| **Rollback after a sale.** A seller is paid, then the block is rolled back. | Owner changes roll back with the block. Off-chain sales SHOULD wait for confirmations; an atomic sale inside the contract avoids the issue. |
| **Phishing.** A user signs a message that transfers a contract without understanding it. | Transfers are messages to the contract. A standard message format (a future SRC) lets wallets decode and show them clearly. |
| **Self-ownership or ownership cycles.** An AT owns itself, or two ATs own each other. | Allowed on purpose; it locks ownership. Explorers SHOULD mark such contracts. |
| **Spam of owner changes.** | At most one owner change per AT and block is written; each call costs steps. |

## Test Cases

| Case | Expected |
| --- | --- |
| AT created after activation, `GET_OWNER` with B2 = 0 | Creator |
| `SET_OWNER` with B1 = existing account | Returns 1; `GET_OWNER` returns B1 in the same run; other ATs see B1 from the next block |
| `SET_OWNER` with B1 = unknown ID | Returns 0; owner unchanged; AT keeps running |
| `SET_OWNER` with B1 = 0 | Returns 0; owner unchanged |
| `SET_OWNER` with B1 = account created in the current block | Returns 0; owner unchanged |
| `SET_OWNER` with B1 = another AT or the executing AT | Returns 1; owner set |
| `SET_OWNER` with B1 = current owner | Returns 1; nothing changes |
| Two successful `SET_OWNER` calls in one run | The second value is stored |
| Block with an owner change is rolled back | Previous owner restored |
| `GET_OWNER` with B2 = AT created before activation | Its creator |
| `GET_OWNER` with B2 = ID that is not an AT | 0 |
| AT created before activation calls 0x030f or 0x0413 | Returns 0; nothing changes |
| `getOwnerContracts` for an account | Lists ATs it owns, including pre-activation ATs it created; not ATs it created but transferred |

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
