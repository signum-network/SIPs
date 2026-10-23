---
sip: 57
title: Reject payments to dead smart contracts
description: Transactions and AT payments to a dead smart contract are rejected instead of being burned
author: ohager
discussions-to: https://github.com/signum-network/signum-node/issues/1064
status: Draft
type: Standard
category: Core
created: 2026-10-21
---

## Abstract

From a new fork height, transactions whose recipient is a dead smart contract (AT) are invalid, and payments that other ATs or subscriptions would make to a dead AT are not executed. Today such payments re-activate the dead AT, which immediately dies again and burns the whole payment. Together with a bug fix in the same fork that stops dead ATs from running at all, no more value can be lost to a contract after it has died.

## Motivation

When an AT hits an error (overflow, invalid opcode, division by zero) and has no error handler, it is marked dead and its remaining balance is burned. A dead AT is not excluded from later runs, though. Its program counter stays on the failing opcode, so any later payment that activates it makes it run again, fail at the same opcode, and burn the whole payment. A dead AT is a black hole for everybody who does not check its state before paying.

A survey of mainnet at height 1,596,584 found:

- 19 dead ATs out of 5,096 non-NFT contracts, each with a different code hash. Most are old test contracts; one is a production DEX contract whose 962 siblings with the same code are still alive.
- About 808,159 SIGNA burned at the moment of death, 808,000 SIGNA of it by that single DEX contract.
- 9 payments to 5 ATs **after** their death, about 99 SIGNA in total, all burned (or paid to the forger before fees were burned). One AT was hit 607,000 blocks after it died.

The losses after death are small so far, but they hit users who cannot know what happened, and nothing prevents larger ones. A wallet can check the `dead` flag of `getATDetails`, but the protocol should not depend on every client doing so.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

### Definitions

- An AT is **dead** if the `dead` flag of its latest machine state is set.
- **Dead AT account**: the account of a dead AT.

### Companion bug fix (same fork, not part of this SIP)

From the fork height, nodes MUST NOT run dead ATs anymore. This is tracked as a bug fix in [signum-node#1063](https://github.com/signum-network/signum-node/issues/1063) and activates at the same height. Without it, this SIP would only reduce the losses. With it, a dead AT stays dead for good and its `dead` flag is never reset.

### Transactions to dead ATs

From the fork height, a transaction MUST be rejected, both when it enters the unconfirmed pool and when it is validated in a block, if:

- its recipient is a dead AT account, or
- it is a multi-out payment (same or different amounts) and any of its recipients is a dead AT account. The whole transaction is rejected.

This applies to all transaction types with a recipient, including messages without amount, asset and multi-asset transfers, and the creation of escrows and subscriptions. The node SHOULD return a specific error that tells the sender the recipient is a dead smart contract.

### Payments created by the protocol

These payments have no sender who could be told that they failed. From the fork height:

- **AT payments**: payments, asset transfers and messages that an AT creates for a dead AT account MUST NOT be executed. The amount and assets stay with the sending AT. The sending AT is not notified.
- **Subscriptions**: a subscription payment due to a dead AT account MUST NOT be made, and the subscription MUST be cancelled.

### Unchanged

- What happens at the moment of death: the remaining balance is still burned. [SIP-56](./sip-56.md) reports the burned part separately from the step fee.
- **Distributions to asset holders** (the distribution transaction and the AT function `distToHolders`) still include dead ATs that hold the asset. Their share stays locked in the dead AT, because it no longer runs. It is not burned.
- ATs that are already dead at the fork height stay dead and are treated like any other dead AT.

## Rationale

### Reject instead of lock or burn

A dead AT never runs again after the companion fix, so anything sent to it would be locked forever. Rejecting the transaction is the only option where nothing is lost, and the sender learns immediately why the payment failed.

### Keep AT payments at the sender

An AT that pays a dead AT cannot handle an error, so the payment can either be dropped or delivered and locked. Keeping it with the sender means no value disappears, and the sending contract can still use the funds, for example to pay someone else later. The downside is that the sending contract's internal bookkeeping may assume the payment went out. Contracts that care can query the state of the recipient before paying, as clients do today.

### Why distributions are not changed

Excluding dead ATs from distributions would change the holder set, and with it the shares of all other holders and the step costs of `distToHolders`. The amounts that dead ATs receive this way are a small share of a distribution. Locking them is simpler and still better than burning them, as happens today.

### Burn at death stays

Burning removes the remaining balance from the supply for good and does not leave a locked balance that might later lead to calls for a rescue fork. Contracts can avoid the burn by setting an error handler (`ERR_ADR`). Compilers SHOULD warn when a contract that holds user funds has none.

## Backwards Compatibility

This is a hard-forking change and must be activated at a fork height, together with the companion bug fix.

- Transactions to dead ATs that are valid today become invalid. Wallets SHOULD check the `dead` flag before creating a transaction and show the new error to the user.
- Contracts that pay other contracts may keep funds they expected to have sent. This only happens when the recipient is dead.
- Existing subscriptions to dead ATs are cancelled at their next payment.

## Test Cases

Before the fork height, current behavior is unchanged. From the fork height:

- A payment, message, asset transfer or multi-asset transfer to a dead AT is rejected in the unconfirmed pool and in a block.
- A multi-out payment with one dead AT among its recipients is rejected entirely.
- Creating an escrow or a subscription with a dead AT as recipient is rejected.
- An AT payment to a dead AT is not executed. The sending AT keeps the amount and assets.
- A subscription payment to a dead AT is not made and the subscription is cancelled.
- A distribution to the holders of an asset that a dead AT holds credits the dead AT, which does not run.
- A dead AT does not run again, whatever its balance (companion fix).
- An AT that dies in block h makes transactions to it invalid from block h+1. Such transactions in the unconfirmed pool are dropped.

## Reference Implementation

Tracked in [signum-node#1064](https://github.com/signum-network/signum-node/issues/1064), the companion fix in [signum-node#1063](https://github.com/signum-network/signum-node/issues/1063). Implementation starts after the community discussion. Affected components in signum-node:

- `FluxValues` / `HistoricalMoment`: fork entry, shared with the companion fix.
- `TransactionType` validation (payments, multi-out, assets, escrow, subscriptions): reject dead AT recipients.
- `AtController.makeTransactions` and `AT.HandleATBlockTransactionsListener`: skip AT payments to dead ATs.
- `SubscriptionServiceImpl`: skip and cancel.
- A cached set of dead AT ids, see below.

## Security Considerations

### Validation cost

Every transaction with a recipient needs a check whether the recipient is a dead AT. Reading the AT state for each transaction would add database load to validation. Nodes SHOULD keep the ids of dead ATs in memory. The set is small (19 on mainnet today) and only grows when an AT dies. It MUST be updated when blocks are applied and rolled back.

### Griefing

Somebody who can make an AT die, for example by sending it data that triggers an unhandled error, can already destroy its balance today. This SIP does not make that easier. After the fork, it can no longer be used to make later deposits of other users disappear.

### Funds that stay locked

Distributions can still credit dead ATs, and these funds stay locked. This is a deliberate trade-off, see Rationale. The locked amounts are visible as the balance of dead ATs and should be reported separately by explorers and supply statistics.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
