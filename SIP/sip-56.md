---
sip: 56
title: Smart contract fee history
description: Optionally record the execution fee of every smart contract run and expose it via API
author: ohager
discussions-to: https://github.com/signum-network/signum-node/issues/1062
status: Draft
type: Standard
category: Interface
created: 2026-10-21
---

## Abstract

This SIP adds an optional, node-local history of smart contract (AT) execution fees. For each AT run, the node records the steps, the fee deducted and how the run ended. The records are written as one batched insert per block, and a new API endpoint `getATFeeHistory` returns them. The feature is disabled by default and is controlled by a node property. It does not change consensus.

## Motivation

The fee an AT pays per run is simple to compute (`steps × step fee`), but it is hard to predict and impossible to look up afterwards:

- **Data-dependent steps:** the number of steps depends on the incoming transactions and the branches the contract takes.
- **Runtime-dependent API costs:** `distToHolders` costs one step per holder. The holder count is determined at runtime, and the distribution only happens if the per-block indirects budget, shared by all ATs in the block, is not exhausted.
- **Dead contracts:** when an AT dies, its whole remaining balance is charged as fee and burned. Users perceive this as a huge gas fee, and the burn cannot be told apart from the step fee. On mainnet, one contract burned 808,000 SIGNA this way (see [SIP-57](./sip-57.md)).
- **No record:** the node deducts the fee from the AT balance, but does not store it anywhere. Today the fee can only be reconstructed from balance differences, mixed with incoming and outgoing payments.

Contract developers, wallets and explorers have no reliable way to answer "what does a run of this contract cost?" or "why did this contract's balance drop?". A per-run fee history gives them minimum, average and maximum costs per contract, makes the activation amounts of contracts easier to choose, and makes dead or frozen contracts visible.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

### Node property

| Property | Type | Default |
|---|---|---|
| `node.atFeeHistory` | boolean | `false` |

- If disabled, the node MUST NOT record fee history and MUST NOT keep any additional per-run data in memory for it.
- The property MAY be enabled on a node that is already synced. Recording then starts at the next applied block, and earlier blocks are not backfilled (see [Recording coverage](#recording-coverage)).

### Recorded data

For each AT executed in an applied block, the node MUST record one entry:

| Field | Description |
|---|---|
| `at` | AT id |
| `height` | height of the block the AT ran in |
| `steps` | steps consumed in this run |
| `fee` | fee for the consumed steps (`steps × stepFee`), in NQT |
| `burned` | remaining balance burned because the AT died, in NQT. `0` unless `result` is `DEAD` |
| `result` | how the run ended, see below |

`fee + burned` is the total amount deducted from the AT in this run. Today the protocol charges both as fee; this SIP only reports them separately and does not change the amounts.

`result` MUST be one of:

| Value | Meaning |
|---|---|
| `FINISHED` | the run ended normally (`FIN`) |
| `STOPPED` | the run stopped (`STP`) and continues at the next activation |
| `FROZEN` | the balance was not sufficient for the next step |
| `MAX_STEPS` | the maximum steps per block were reached |
| `DEAD` | the AT died, its remaining balance was burned (see `burned`) |

ATs that are skipped in a block (sleeping, frozen on same balance, insufficient balance before the first step) MUST NOT be recorded.

### Writing

- All entries of one block MUST be written in the same database transaction as the block itself.
- Entries SHOULD be written with one batched statement per block, so the number of statements does not depend on the number of ATs in the block.
- The storage MUST NOT be versioned: each entry is written once and never updated.

### Rollback

When blocks are popped off, the node MUST delete all entries above the new chain height. The deletion MUST happen even if recording is currently disabled, so that a later re-enabling never mixes entries from an abandoned fork with new ones.

### Recording coverage

Because recording can be switched on and off, the history may contain gaps. The node MUST track the height `recordedFrom` from which the history is complete up to the current chain height:

- When recording starts (node start with the property enabled), and the last block was not recorded, `recordedFrom` MUST be set to the height of the next applied block.
- After a rollback below `recordedFrom`, `recordedFrom` MUST be set to the height of the next applied block.
- Entries below `recordedFrom` MAY be kept. They are valid but may be incomplete.

### API

New endpoint `getATFeeHistory` (GET):

| Parameter | Required | Description |
|---|---|---|
| `at` | yes | AT id |
| `heightStart` | no | lowest height to include |
| `heightEnd` | no | highest height to include |
| `firstIndex`, `lastIndex` | no | pagination, as in other list endpoints |

Response:

```json
{
  "at": "1234567890",
  "recordedFrom": 1250000,
  "stepFee": "100000",
  "entries": [
    { "height": 1250123, "steps": 1834, "fee": "183400000", "burned": "0", "result": "FINISHED" }
  ]
}
```

- Entries MUST be sorted by height, newest first.
- `stepFee` is the current step fee of this AT's version, in NQT.
- If the feature is disabled and no entries are stored, the node MUST return error code 9 (`Feature not available`).
- If the feature is disabled but older entries exist, the node SHOULD return them and MUST omit `recordedFrom`.
- Public nodes SHOULD expose whether the feature is enabled, for example in `getState` or `getConstants`, so that clients can pick a node that records it.

## Rationale

### Per-run entries instead of aggregates

An aggregate per AT (run count, total and maximum fee) would need much less disk space, but it means a read-modify-write for every AT in every block. AT processing is one of the hotspots during synchronization, and the node already caches AT data to reduce database access. An append-only insert is cheaper than an update, and batching it per block keeps the statement overhead constant. Aggregates can be computed from the entries, but not the other way around.

### Optional and node-local

The history does not influence consensus and is not needed to validate blocks. Making it optional means that nodes which do not need it, such as mining and private nodes, pay no storage or sync cost. Archive and public nodes that serve explorers and wallets can enable it.

### Accepting gaps

Requiring a complete history would force a full resync whenever the feature is enabled. Allowing gaps and reporting `recordedFrom` lets operators enable it on a synced node, while clients can still tell which part of the history is reliable.

### Not versioned

AT state and map tables are versioned and trimmed after the rollback window, which costs an update per write. A fee entry never changes after it is written, so versioning is not needed. Rollback is a plain delete by height.

## Backwards Compatibility

This SIP does not change consensus, the block format or existing API responses. Nodes with and without the feature are fully compatible. The only change for existing installations is a new, empty database table created by a migration.

## Test Cases

- With the feature disabled, applying blocks writes no entries and the endpoint returns error code 9.
- With the feature enabled, a block with N executed ATs produces exactly N entries, written in one statement.
- Skipped ATs (sleeping, frozen on same balance) produce no entry.
- `result`, `fee` and `burned` are correct for each end state. For `DEAD`, `fee` covers only the steps and `burned` holds the remaining balance; `fee + burned` equals the amount deducted from the AT.
- Popping off blocks deletes the entries above the new height, also when the feature is disabled.
- Enabling the feature on a synced node sets `recordedFrom` to the next block. Restarting with the feature still enabled keeps `recordedFrom`.
- A rollback below `recordedFrom` moves `recordedFrom` to the next applied block.
- Sync benchmark: syncing a height range with many AT runs, with and without the feature, to quantify the overhead.

## Reference Implementation

Tracked in [signum-node#1062](https://github.com/signum-network/signum-node/issues/1062). Implementation starts after the community discussion. Affected components in signum-node:

- `AtController`: return the run result from `runSteps` and pass steps and result along with the fee.
- `AT.addPendingFee` / `pendingFeesMap`: carry steps and result. `HandleATBlockTransactionsListener`: write the batch for the block.
- Flyway migrations for SQLite, MariaDB and PostgreSQL: table `at_fee(at_id, height, steps, fee, burned, result)` with an index on `(at_id, height)`.
- A store for writing, querying and rollback, the property in `Props`, and the API handler.

## Security Considerations

### Storage growth

The history grows without limit while it is enabled. Each entry takes about 60–70 bytes including the index, so the size is roughly `AT runs per block × 131,400 blocks per year × 65 bytes`. With 10 runs per block, that is about 90 MB per year. Operators enabling the feature SHOULD monitor disk usage. Pruning by age MAY be added later without changing this specification.

Because the feature is disabled by default, an attacker who deploys many cheap ATs cannot increase the storage of nodes that do not record history. On nodes that do record it, every recorded run was paid for with at least one step fee.

### Sync performance

Writing entries adds work to block processing. Entries are written in the block's transaction, so there is no extra commit, and with a batched insert there is one statement per block. Implementations MUST NOT read from the history while applying blocks.

### Trust in reported data

The history is node-local and not part of consensus. A malicious or misconfigured node can report wrong values. Clients that need certainty SHOULD compare several nodes or verify against balance changes on chain.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
