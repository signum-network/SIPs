---
sip: 54
title: Delete map entries in smart contracts
description: Let smart contracts delete their own map entries so that unused entries stop occupying node storage
author: ohager
discussions-to: https://github.com/signum-network/signum-node/issues/1060
status: Draft
type: Standard
category: Core
created: 2026-10-21
requires: 38
---

## Abstract

This SIP adds a new AT API function `DELETE_MAP_VALUE_KEYS_IN_A` (`0x0412`). With it, a smart contract explicitly deletes entries from its own map ([SIP-38](./sip-38.md)). Today an entry can only be set to `0`, and the row stays in the node database forever. The function activates at a new fork height. The SIP also describes non-normative node changes that physically remove deleted entries once they are outside the rollback window.

## Motivation

[SIP-38](./sip-38.md) introduced `GET_MAP_VALUE_KEYS_IN_A` (`0x0407`) and `SET_MAP_VALUE_KEYS_IN_A` (`0x0408`). A contract can only "clear" an entry by setting it to `0`:

- the row stays in the database (`latest=true, value=0`)
- setting `0` on a key that never existed even inserts a new row

Contracts with many short-lived entries (orders, sessions, temporary participants) make the map table grow without bound, although the data is no longer needed.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

### New operation

- `DELETE_MAP_VALUE_KEYS_IN_A = 0x0412` (`EXT_FUN`): deletes the entry with keys in A1 and A2 from the contract's own map.

Rules:

- Only the contract's own map can be modified, as with `SET_MAP_VALUE_KEYS_IN_A`.
- Deleting a non-existent key MUST be a no-op.
- The function costs the standard `apiStepMultiplier` (10 steps), like every API function.
- `GET_MAP_VALUE_KEYS_IN_A` MUST return `0` for a deleted key, the same as for a key that never existed.
- Within one execution, the last operation on a key wins:
  - set → delete: the key ends up deleted, and a subsequent read returns `0`
  - delete → set: the key ends up with the new value

### Activation

- The function activates at a new fork height.
- Before the fork height, `0x0412` MUST remain a no-op (unknown functions return `0`), so a contract that calls it before the fork behaves the same before and after.

## Rationale

- **Explicit delete instead of "set to `0` means delete":** a stored `0` remains a real value, and the node API can distinguish "set to 0" from "absent". Contracts see no difference, because both read as `0`.
- **Versioned delete:** entries are deleted with the node's existing versioned delete (a tombstone row), so chain rollbacks keep working.

## Backwards Compatibility

This is a hard-forking change and breaks compatibility with old fully-validating nodes. It MUST NOT be deployed without widespread consensus.

- Existing contracts are not affected. Setting `0` keeps its current behavior.
- API clients reading map values (`getATMapValues`) will no longer see deleted entries.

## Test Cases

- Delete an existing key; a subsequent read returns `0`.
- Delete a non-existent key is a no-op and creates no row.
- set → delete and delete → set within one execution.
- A contract cannot delete entries of another contract's map.
- Rollback across a delete restores the previous value.
- Before the fork height, `0x0412` is a no-op.

## Reference Implementation

Tracked in [signum-node#1060](https://github.com/signum-network/signum-node/issues/1060). Affected components: `OpCode`, `AtApiController` / `AtApiPlatformImpl`, `AtMachineState` (deleted marker for pending map updates), `AT.saveMapUpdates`, `FluxValues`.

### Non-normative: storage cleanup in the node

These changes are local database maintenance. Block validity never depends on them, so they need no fork and can ship independently:

- **Trim removes deleted entities:** today the versioned tables keep the tombstone row of a deleted entity forever, because `trim()` only removes older versions of a key. Trim SHOULD also remove all rows of a key whose newest row is a tombstone below the trim height. No rollback can reach those rows. This benefits all versioned tables, for example filled orders, zero-balance asset rows, subscriptions and escrows.
- **One-time migration of existing zero values:** existing map rows with `value=0` are live rows, and deployed contracts cannot call the new function. A database migration SHOULD mark them as deleted (`latest=false`) instead of deleting them, so rollbacks stay correct. The next trim then removes them. The migration MUST ship in the same release as the trim change.

Tooling: SmartC SHOULD expose a built-in such as `deleteMapValue(k1, k2)`. The SmartC testbed/simulator SHOULD support map deletion.

## Security Considerations

- **Isolation:** a contract MUST only delete entries of its own map. Otherwise contracts could destroy other contracts' data.
- **Rollback safety:** deletes use the existing versioned delete, so reorganizations restore deleted entries correctly. The non-normative cleanup only removes rows below the trim height, which no rollback can reach.
- **Fork safety:** activation by block height prevents contracts that already call `0x0412` from changing behavior retroactively.
- **Cost:** a delete costs the same as a set, so it creates no new spam vector. It reduces node storage.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
