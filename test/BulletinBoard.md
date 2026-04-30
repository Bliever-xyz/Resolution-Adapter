# BulletinBoard.t.sol — Test Documentation

## Purpose

Tests the `BulletinBoard` abstract mixin inherited by `BlieverUmaAdapter`. BulletinBoard provides a permissionless, per-question on-chain update registry used by UMA DVM voters to receive clarifications from question creators.

Tests are exercised through the deployed `BlieverUmaAdapter` proxy (since `BulletinBoard` is an abstract contract with no independent deployment).

---

## Contract Under Test

`contracts/src/mixins/BulletinBoard.sol` (via `BlieverUmaAdapter`)

---

## Mixin Summary

```
postUpdate(bytes32 questionId, bytes update)
    → stored under keccak256(abi.encode(questionId, msg.sender))
    → emits AncillaryDataUpdated

getUpdates(bytes32 questionId, address owner) → AncillaryDataUpdate[]
    → returns all updates posted by `owner` for `questionId`

getLatestUpdate(bytes32 questionId, address owner) → AncillaryDataUpdate
    → returns the last element; zero struct if none
```

---

## Test Groups

### 1. `postUpdate` — Basic Behavior

| Test | What it verifies |
|------|-----------------|
| `test_postUpdate_anyAddressCanPost` | Attacker, alice, factory can all post without restriction |
| `test_postUpdate_emitsAncillaryDataUpdatedEvent` | `AncillaryDataUpdated(questionId, poster, update)` emitted |
| `test_postUpdate_storesCorrectData` | Retrieved bytes match what was posted |
| `test_postUpdate_storesTimestampCorrectly` | Each update's `timestamp` matches `block.timestamp` at the time of posting |

---

### 2. `getUpdates` — Array Retrieval

| Test | What it verifies |
|------|-----------------|
| `test_getUpdates_emptyArrayBeforeAnyPost` | Returns empty array when no updates exist |
| `test_getUpdates_orderedAfterMultiplePosts` | Three updates by alice returned in posting order (A, B, C) |
| `test_getUpdates_posterIsolation` | Alice and Bob have independent arrays for the same questionId |
| `test_getUpdates_separatePerQuestionId` | Same poster has independent arrays for different questionIds |

---

### 3. `getLatestUpdate` — Single Latest Entry

| Test | What it verifies |
|------|-----------------|
| `test_getLatestUpdate_zeroStructWhenNoUpdates` | Returns `AncillaryDataUpdate{timestamp: 0, update: ""}` when empty |
| `test_getLatestUpdate_returnsMostRecentUpdate` | After three posts, returns the third (UPDATE_C) |
| `test_getLatestUpdate_singlePost` | Works correctly with only one post |

---

### 4. Fuzz Tests

| Test | Property |
|------|----------|
| `testFuzz_postUpdate_roundTrip` | Any bytes payload of length `[1, 1000]` can be posted and retrieved without corruption |

---

## Key Observations

- **Permissionless:** No role check in `postUpdate`. Anyone can post updates for any `questionId`. The protocol instructs DVM voters to weight updates from the question creator (`qd.creator`).
- **Mapping key:** `keccak256(abi.encode(questionId, msg.sender))` — ensures per-poster isolation. Two different posters for the same question never interfere.
- **No deletion:** Updates are append-only. `getLatestUpdate` is the UI-friendly accessor for the most recent clarification.
- **No relation to resolved state:** The bulletin board is independent of the oracle lifecycle; posts can be made before or after resolution.
