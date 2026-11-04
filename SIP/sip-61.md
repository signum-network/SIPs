---
sip: 61
title: Smart contract alarm
description: A stopped smart contract wakes up at a set block height, while incoming transactions still wake it as before
author: ohager
discussions-to: https://github.com/signum-network/signum-node/issues/1076
status: Draft
type: Standard
category: Core
created: 2026-11-04
---

## Abstract

This SIP gives every smart contract (AT) one alarm slot. A new AT API function `SET_ALARM` (0x0414) sets a block height at which a stopped AT runs again, even if it received nothing. Until then, incoming transactions wake the AT exactly as today. `GET_ALARM` (0x0310) reads the current alarm.

The alarm fires once and then clears itself. Setting it to 0 disables it. Unlike `sleep`, the alarm never blocks the AT: it only adds a wake-up condition. The node stores the alarm height in the AT state and reports it as `alarmHeight`.

## Motivation

Many contracts must act at a certain time and also react to transactions in the meantime: a game round ends, an auction closes, a timeout refunds a deposit. Today an AT can do only one of the two.

| Instruction | Effect | Do incoming transactions wake the AT? |
| --- | --- | --- |
| `sleep n` (`SLP_DAT`) | The AT is blocked for n blocks, then runs | No. Transactions wait until the sleep ends. |
| `stop` / `finish` (`STP`, `FIN`) | The AT is frozen until it receives at least its activation amount | Yes. Without a transaction it never runs again. |

Contracts work around this in two ways:

- **Helper contract.** A second AT sleeps and then sends a message to the main AT to wake it. Each must know the other's ID, which is not known before deployment. The creator sets the IDs after deploying both, which is a circular dependency and an extra trust step. The community reported this for the game Stellar Duel.
- **Polling.** The AT sleeps one block, checks for transactions and the height, and sleeps again. It pays for a run in every block, even when nothing happens.

An alarm removes both workarounds: the AT stops as usual, wakes up on transactions, and wakes up on its own at the set height.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

In this section, H is the height of the block in which the AT runs. Inside the AT, it is the height part of `GET_BLOCK_TIMESTAMP` (0x0300).

### When an AT runs today

An AT runs in block H only if all three conditions hold:

1. Its next height is at most H. `sleep n` sets the next height to H + n.
2. Its balance covers the minimum fee for one run.
3. It is not frozen, or its balance grew by at least its minimum activation amount since its last run. `stop` and `finish` freeze the AT.

### Alarm

- Every AT has one alarm slot that holds a block height. 0 means no alarm. A new AT starts with 0.
- Condition 3 becomes: the AT is not frozen, **or** its balance grew by at least its minimum activation amount, **or** its alarm is not 0 and at most H.
- Conditions 1 and 2 stay unchanged. A sleeping AT does not run on its alarm, and neither does an AT without enough balance. The alarm stays set and fires in the first later block in which the AT runs.
- When the AT runs in a block with an alarm that is not 0 and at most H, the node MUST clear the alarm to 0 before the run starts. This holds whatever woke the AT: the alarm, a transaction, or the end of a sleep.
- An AT that is not frozen, for example after a sleep, runs as today. The alarm adds a wake-up condition; it never prevents a run.

### Function SET_ALARM

| Property | Value |
| --- | --- |
| Name | `SET_ALARM` |
| Function number | 0x0414 |
| Opcode | `EXT_FUN_DAT` (0x33) |
| Assembly | `FUN 0x0414 $addr` |
| Input | `$addr` = alarm height (absolute block height); 0 = no alarm |
| Output | none |
| Registers | A and B unchanged |

The node stores the input value v as follows:

| Input v | Stored alarm |
| --- | --- |
| v ≤ 0 | 0 (alarm disabled) |
| 1 ≤ v ≤ H | H + 1 (next block) |
| H < v ≤ H + max | v |
| v > H + max | H + max |

max is the existing limit for `sleep` (`getMaxWaitForNumOfBlocks`, currently 31,536,000 blocks).

Rules:

1. `SET_ALARM` MUST only change the alarm of the executing AT.
2. A new call replaces the previous alarm. If an AT calls `SET_ALARM` several times in one run, the last call wins.
3. The call MUST NOT raise an error or stop the AT, whatever the input.
4. The alarm is saved with the AT state at the end of the run. A rollback of the block MUST restore the previous alarm.

### Function GET_ALARM

| Property | Value |
| --- | --- |
| Name | `GET_ALARM` |
| Function number | 0x0310 |
| Opcode | `EXT_FUN_RET` (0x35) |
| Assembly | `FUN @addr 0x0310` |
| Input | none |
| Output | `@addr` = the alarm height of the executing AT; 0 if no alarm is set |
| Registers | A and B unchanged |

`GET_ALARM` returns the value as stored by the last `SET_ALARM` in the same run, or the value from the start of the run. In a run that cleared the alarm, it returns 0 until the AT sets a new one.

### Cost

Both functions cost the standard `API_STEP_MULTIPLIER` (10) steps. The run that an alarm triggers costs the normal steps, paid from the AT's balance.

### Block validation

Validating nodes MUST accept a run of an AT that is frozen and received no activation amount if its alarm is not 0 and at most H. They MUST reject such a run if the alarm is 0 or greater than H.

### Node API

All responses that describe an AT (`getAT`, `getATs`, `getAccountATs`, `getATDetails`) MUST add the field `alarmHeight`: the stored alarm, 0 if none.

### Activation

A new flux value at the next hard fork enables the alarm. `SET_ALARM` and `GET_ALARM` MUST be available only to ATs whose creation height is at or after the activation height.

For all other ATs, a call MUST behave as today: an unknown function changes nothing, and an unknown `EXT_FUN_RET` function sets `@addr` to 0. Their alarm stays 0, so their scheduling does not change.

## Rationale

- **Wake-up condition instead of a new sleep mode.** The scheduler already wakes frozen ATs on incoming balance. The alarm adds one more condition to the same check, so the AT lifecycle does not change.
- **Absolute height.** One value is enough and it is unambiguous. A contract that needs a relative delay computes H + n. Setting the same height again in a later run changes nothing, which a relative delay would.
- **Fires once and clears itself.** A cleared alarm cannot drain the AT. A permanent alarm in the past would start the AT in every block until it runs out of balance. A contract that needs a repeating timer sets the next alarm in each run.
- **Cleared on any run at or after the alarm.** The contract sees one run, not two, when a transaction and the alarm arrive together. It detects an alarm run by comparing H with the height it stored when setting the alarm.
- **Sleep has priority.** `sleep` is an explicit request not to run. An alarm that overrides it would change the meaning of existing code patterns.
- **The alarm survives a lack of balance.** A contract that is topped up later still performs its timed action. Dropping the alarm would leave it waiting for a transaction that may never come.
- **Clamping instead of failing.** A past height means "as soon as possible", so the next block. The upper limit is the one that `sleep` already uses. Neither case stops the contract.
- **One slot.** It covers the reported use cases and keeps the state at one integer per AT. Contracts with several deadlines set the alarm to the earliest one.
- **No other AT can set or read the alarm.** The alarm is internal scheduling state. Other contracts have no use for it, and writing it from outside would let one AT make another pay for runs.
- **Only new ATs.** Same rule as [SIP-58](./sip-58.md), [SIP-59](./sip-59.md) and [SIP-60](./sip-60.md): no deployed contract changes behaviour.
- **Function numbers.** 0x0310 is the next free number in the 0x03 group (AT information), after `GET_OWNER` (0x030f, [SIP-60](./sip-60.md)). 0x0414 is the next free number in the 0x04 group (state changes), after `SET_OWNER` (0x0413, [SIP-60](./sip-60.md)).
- **Storage (non-normative).** A column `alarm_height` (integer, default 0) in the versioned table `at_state` is RECOMMENDED. It is written with the rest of the AT state, rolls back with it, and lets the scheduler query select alarmed ATs with one more condition.

## Backwards Compatibility

Existing ATs are not affected. A pre-activation AT that calls 0x0310 or 0x0414 gets the same result as today, and its alarm stays 0. The scheduling of existing ATs does not change. API responses gain one field; clients that ignore unknown fields need no change.

The helper-contract and polling patterns keep working. New contracts can replace them with an alarm.

### Tooling

SmartC needs built-ins, for example `setAlarm(height)` and `getAlarm()`. SignumJS, the AT API documentation and explorers need the new field. Tooling is out of scope for this SIP.

## Security Considerations

The alarm sets the earliest height of a run, not an exact one. Contracts MUST NOT assume that an alarm run happens exactly at the alarm height.

| Threat | Mitigation |
| --- | --- |
| **Delayed alarm.** The AT runs later than its alarm: the block is full, the generator leaves the AT out, the AT sleeps or lacks balance. | Same as any AT run today. The alarm stays set until the AT runs. Contracts read H in the alarm run instead of assuming the alarm height. |
| **Many alarms in one block.** Many ATs set the same height. | The AT payload per block is already limited. ATs that do not fit run in later blocks. Each run is paid by its AT, so mass alarms cost their creators. |
| **Draining the AT.** An alarm keeps starting the AT. | The alarm clears itself when it fires. Only the contract's own code can set a new one, and each run is paid from its balance. |
| **Another AT sets or clears the alarm.** | `SET_ALARM` and `GET_ALARM` act only on the executing AT. |
| **Contract stopped by a bad input.** | Every input is clamped to a valid value. `SET_ALARM` never raises an error or stops the AT. |
| **Late transactions in the alarm run.** A user sends a transaction in the block of the deadline, and the alarm run processes it together with the deadline action. | The contract compares each transaction's height with its deadline and treats later ones as late. This is contract logic, as with today's helper contracts. |
| **Rollback.** A block with an alarm run or a new alarm is rolled back. | The alarm is part of the versioned AT state and rolls back with it. |

## Test Cases

| Case | Expected |
| --- | --- |
| New AT, `GET_ALARM` | 0 |
| `SET_ALARM` with H + 10, then `finish`; no transactions | AT runs at H + 10; alarm cleared; `GET_ALARM` returns 0 in that run |
| `SET_ALARM` with H + 10, then `finish`; transaction with activation amount at H + 3 | AT runs at H + 3 and at H + 10 |
| Transaction with activation amount arrives in the alarm block | One run; alarm cleared |
| `SET_ALARM` with 0 or a negative value | Alarm 0; AT wakes only on transactions |
| `SET_ALARM` with a height ≤ H | Alarm H + 1 |
| `SET_ALARM` with a height above H + max | Alarm H + max |
| Two `SET_ALARM` calls in one run | The second value is stored; `GET_ALARM` returns it in the same run |
| `SET_ALARM` with H + 5, then `sleep 10` | AT does not run at H + 5; runs at H + 10; alarm cleared |
| Alarm reached, balance below the minimum fee | AT does not run; alarm stays; AT runs in the first block after it is funded |
| Alarm reached, block payload full | AT runs in a later block; alarm cleared then |
| Block with an alarm run is rolled back | Previous alarm restored |
| Validator receives a block with a frozen AT, no activation amount, alarm 0 or above H | Block rejected |
| AT created before activation calls 0x0310 or 0x0414 | Same result as today; alarm stays 0 |

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
