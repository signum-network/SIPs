---
sip: 58
title: SHA256 over AT memory ranges
description: Smart contracts can hash an arbitrary byte range of their data segment with SHA256
author: ohager
discussions-to: https://github.com/signum-network/signum-node/issues/1073
status: Draft
type: Standard
category: Core
created: 2026-10-23
---

## Abstract

This SIP adds one AT API function, `SHA256_RANGE_TO_B` (`0x0207`), that hashes an arbitrary byte range of the AT data segment with SHA256 and writes the digest to register B. Today an AT can only hash exactly 32 bytes from register A. The new function lets contracts hash inputs up to the full data segment (2560 bytes), at a step cost that grows with the input length.

## Motivation

The existing hash functions `SHA256_A_To_B` (`0x0204`) and `CHECK_SHA256_A_WITH_B` (`0x0205`) take exactly 32 bytes from register A. That is enough for HTLC preimages, but useless for longer data: SHA256 is not composable, so `SHA256(a || b)` cannot be built from `SHA256(a)` and `SHA256(b)`.

**Primary use case: Bitcoin SPV proofs inside an AT.** Collateral-based P2P trading (BTDEX-style) currently needs human mediators when a buyer and seller disagree on whether a BTC payment happened. With this function, the buyer can instead submit a proof that the AT verifies on its own:

- Block header: 80 bytes, hashed with SHA256d (this function, then `SHA256_A_To_B`).
- Merkle step: 64 bytes (left and right node concatenated), one call per tree level.
- Transaction id: SHA256d over the non-witness serialization, typically 110 to 1000 bytes.

The same works for Bitcoin Cash (SHA256d proof of work). It does not cover Litecoin or Dogecoin, whose proof of work uses scrypt.

**Secondary use cases:** commit-reveal schemes with payloads longer than 32 bytes, and checking that data submitted by message matches a known SHA256 digest (for example a document hash).

**History.** An unfinished `sha256ToB(val1, val2)` has existed in `AtApiImpl` since the 2016 Burst import, but it was never callable: its dispatch (`func_num == 1536`) was commented out from the start, and the 2019 refactor removed that comment. The code mixes units (bounds checked in 64-bit words, hashing done in bytes) and silently caps input at 256 bytes. This SIP specifies the function from scratch instead of activating that code.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

**Function.** `SHA256_RANGE_TO_B`, number `0x0207`, called with opcode `EXT_FUN_DAT_2` (`0x34`). In AT assembly: `FUN 0x0207 $start $len`.

**Parameters.**

- `start`: start address in 64-bit words within the data segment. The first hashed byte is at byte offset `start × 8`.
- `len`: number of bytes to hash. It does not need to be a multiple of 8.

**Validity.** The call is valid if and only if all of the following hold, evaluated without integer overflow:

- `start ≥ 0` and `len ≥ 0`
- `start × 8 + len ≤ dSize`, where `dSize` is the data segment size in bytes (data pages × 256)

**Effect of a valid call.** B1..B4 are set to `SHA256(data[start × 8 … start × 8 + len − 1])`, using the same byte layout as `SHA256_A_To_B`: digest bytes 0–7 go to B1 as a little-endian 64-bit value, bytes 8–15 to B2, and so on. Writing B1..B4 back to memory therefore yields the digest bytes in order. `len = 0` is valid and yields the SHA256 of the empty string.

**Effect of an invalid call.** B1..B4 are set to zero. Execution continues; no error is raised and the AT is not stopped.

**Step cost.** `API_STEP_MULTIPLIER + ceil(len / 64)` steps for a valid call; `API_STEP_MULTIPLIER` (currently 10) for an invalid call. At the current step fee of 0.001 SIGNA (100,000 Planck) per step:

| Input | Bytes | Steps | Fee (SIGNA) |
| --- | --- | --- | --- |
| Merkle step | 64 | 11 | 0.011 |
| Bitcoin block header | 80 | 12 | 0.012 |
| Typical Bitcoin transaction | 250 | 14 | 0.014 |
| Full data segment | 2560 | 50 | 0.050 |

**Activation.** The function is enabled by a new flux value at the height of the next hard fork, on mainnet and testnet. It is available only to ATs created at or after that height. For all other ATs, and for every AT before that height, `0x0207` behaves as it does today: an unknown `EXT_FUN_DAT_2` function is a no-op that costs `API_STEP_MULTIPLIER` steps.

## Rationale

- **Start in words, length in bytes.** Every AT address is a word index, so `start` follows that convention. Byte length is needed because real inputs, such as raw Bitcoin transactions, have arbitrary lengths.
- **No fixed cap.** The data segment is already bounded at 10 pages × 256 bytes = 2560 bytes (`MAX_MACHINE_DATA_PAGES`). The legacy 256-byte cap would exclude many ordinary Bitcoin transactions.
- **Cost scales with length.** SHA256 processes 64-byte blocks, so CPU cost is linear in `len`. Hashing 2.5 KB takes microseconds, but tying cost to work keeps the fee model consistent and removes any incentive to hash large ranges for free.
- **B is zeroed on invalid input.** A contract can detect the failure with `Check_B_Is_Zero`, since a valid digest of all zeros is practically impossible. Leaving B unchanged would risk a contract mistaking a stale earlier hash for a fresh result. Zeroing also matches the existing functions that write B: `MESSAGE_FROM_TX_IN_A_TO_B`, `B_TO_ADDRESS_OF_TX_IN_A` and `B_TO_ASSET_IDS_OF_TX_IN_A` all set B to zero for an invalid transaction.
- **No streaming API (init/update/final).** It would make the internal SHA256 state part of the persisted AT state, adding consensus-critical serialization for little gain. The data segment is large enough for the target use cases.
- **No SHA256d or check variant.** Both can be composed from existing functions: `Copy_A_From_B` + `SHA256_A_To_B` for the second round, and `Check_A_Equals_B` for comparison.
- **No other range hashes (RIPEMD160, Keccak).** No concrete use case requires them. Every function number is permanent consensus code.
- **Number `0x0207` instead of the legacy `0x0600`.** It groups the function with the other hash functions (`0x0200`–`0x0206`). `0x0600` was never active in Burst or Signum.

## Backwards Compatibility

Existing functions are unchanged; in particular `SHA256_A_To_B` and `CHECK_SHA256_A_WITH_B` keep their 32-byte behavior. This SIP still requires a hard fork, because AT execution results change after activation.

ATs created before activation are not affected: for them `0x0207` stays a no-op permanently. This rules out any behavior change in deployed contracts, including one deployed shortly before the fork with a call that looks harmless today. No scan of deployed AT code is needed.

The legacy `sha256ToB` method in `AtApi` and `AtApiImpl` is removed. It is unreachable, so removing it has no consensus effect.

## Test Cases

All digests below were verified with `sha256sum` and `openssl`. Input bytes are placed in the data segment starting at a word boundary.

**Digest correctness**

| Input | len | Expected digest (hex, byte order as in memory after the call) |
| --- | --- | --- |
| empty | 0 | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` |
| `abc` (partial word) | 3 | `ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad` |
| `abcdbcdecdefdefgefghfghighijhijkijkljklmklmnlmnomnopnopq` | 56 | `248d6a61d20638b8e5c026930c3e6039a33ce45964ff2167f6ecedd419db06c1` |
| Bitcoin genesis block header | 80 | `af42031e805ff493a07341e2f74ff58149d22ab9ba19f61343e2c86c71c5d66d` |

The genesis header is `0100000000000000000000000000000000000000000000000000000000000000000000003ba3edfd7a7b12b27ac72c3e67768f617fc81bc3888a51323a9fb8aa4b1e5e4a29ab5f49ffff001d1dac2b7c`. Hashing its digest again with `SHA256_A_To_B` must give `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000`, which byte-reversed is the familiar block hash `000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f`.

**Bounds** (B1..B4 must be zero unless marked valid)

- `start × 8 + len = dSize`: valid, hashes up to the last byte.
- `start × 8 + len = dSize + 1`: invalid.
- `start = -1` or `len = -1`: invalid.
- `start = Long.MAX_VALUE`, `len = 1` (overflow): invalid, no exception.
- `start` valid, `len = Long.MAX_VALUE`: invalid, no exception.

**Step cost**

- `len` = 0, 1, 64, 65, 2560 → 10, 11, 11, 12, 50 steps.
- Invalid call → 10 steps.

**Activation**

- Before activation, any AT: `0x0207` is a no-op, B unchanged, 10 steps.
- After activation, AT created at or after activation: the digest is written as specified.
- After activation, AT created before activation: `0x0207` is a no-op, B unchanged, 10 steps.

## Reference Implementation

No new dependency is needed; the node already provides `Crypto.sha256()`.

| File (signum-node) | Change |
| --- | --- |
| `src/brs/at/OpCode.java` | Add `SHA256_RANGE_TO_B = 0x0207` |
| `src/brs/at/AtApiController.java` | Dispatch in `func2`, gated by the new flux value and the AT creation height |
| `src/brs/at/AtApi.java`, `AtApiImpl.java` | New implementation; remove legacy `sha256ToB` |
| `src/brs/at/AtMachineProcessor.java` | Length-based steps in `getNumSteps`, reading `$len` (precedent: `DIST_TO_ASSET_HOLDERS`) |
| `src/brs/fluxcapacitor/` | New flux value and activation heights for mainnet and testnet |
| `test/java/brs/at/` | Unit tests for all test cases above |

Outside the node, the function needs an entry in the AT API documentation. Compiler and tooling support is out of scope for this SIP.

## Security Considerations

### Execution cost

The input is bounded by the data segment (at most 2560 bytes), and the step cost grows with the input length. A single call costs at most 50 steps, so the function adds no new denial-of-service vector. Hashing uses the SHA256 implementation the node already uses for consensus.

### Bounds and overflow

`start` and `len` come from contract memory and can hold any 64-bit value. Implementations MUST check the bounds without integer overflow and MUST NOT throw on invalid input; an invalid call only zeroes B. The test cases include the overflow cases.

### Failure detection

A contract MUST check B for zero after the call before using the digest. Otherwise a wrong `start` or `len` turns into a comparison against an all-zero digest.

### SPV proofs

This function only provides the hashing. A contract that verifies Bitcoin payments still has to check proof of work against a difficulty target, link headers to a trusted checkpoint and require enough confirmations. Those checks are the contract's responsibility and are not part of this SIP.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
