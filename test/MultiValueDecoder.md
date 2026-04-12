# MultiValueDecoder.t.sol — Test Documentation

## Purpose

Full coverage of the `MultiValueDecoder` pure library, which encodes and decodes UMIP-183 `MULTIPLE_VALUES` prices used by `BlieverUmaAdapter` to determine the winning outcome from UMA oracle settlements.

---

## Contract Under Test

`contracts/src/libraries/MultiValueDecoder.sol`

---

## UMIP-183 Encoding Contract

The library interprets a single `int256` as up to 7 packed `uint32` slots:

```
int256 layout (256 bits):
  bits  0– 31  → slot 0 (outcome 0 value)
  bits 32– 63  → slot 1 (outcome 1 value)
  ...
  bits 192–223 → slot 6 (outcome 6 value)
  bits 224–255 → ALWAYS ZERO (collision guard — must not overlap with sentinels)
```

**Winning outcome contract (Bliever-specific):**
- Exactly one slot in `[0, outcomeCount)` must equal `1`.
- All other slots in `[0, outcomeCount)` must equal `0`.
- All slots in `[outcomeCount, 7)` must equal `0` (trailing garbage guard).
- Bits 224–255 must be zero.

**Special sentinels (must be checked BEFORE decoding):**
- `type(int256).min` → `TOO_EARLY_PRICE`
- `type(int256).max` → `UNRESOLVABLE_PRICE`

---

## Test Groups

### 1. `isTooEarly` — Sentinel Detection

| Test | What it verifies |
|------|-----------------|
| `test_isTooEarly_trueForMinInt256` | `type(int256).min` → true |
| `test_isTooEarly_falseForZero` | `0` → false |
| `test_isTooEarly_falseForPositive` | Any positive → false |
| `test_isTooEarly_falseForMaxInt256` | `type(int256).max` is UNRESOLVABLE, not TOO_EARLY |

---

### 2. `isUnresolvable` — Sentinel Detection

| Test | What it verifies |
|------|-----------------|
| `test_isUnresolvable_trueForMaxInt256` | `type(int256).max` → true |
| `test_isUnresolvable_falseForZero` | `0` → false |
| `test_isUnresolvable_falseForMinInt256` | `type(int256).min` is TOO_EARLY, not UNRESOLVABLE |
| `test_isUnresolvable_falseForPositive` | Any positive → false |

---

### 3. `decodeWinningOutcome` — Binary (N=2) All Outcomes

| Test | Encoding | Expected winner |
|------|----------|-----------------|
| `test_decode_binary_outcome0Wins` | `int256(1)` (slot 0 = 1) | 0 |
| `test_decode_binary_outcome1Wins` | `int256(1 << 32)` (slot 1 = 1) | 1 |

---

### 4. `decodeWinningOutcome` — Multi-Outcome (N=7) All Outcomes

| Test | What it verifies |
|------|-----------------|
| `test_decode_sevenOutcome_allOutcomes` | Each of outcomes 0–6 encodes as `int256(1 << (32*i))` and decodes back to `i` |

This is the stress test for the 7-slot UMIP-183 encoding.

---

### 5. Invalid: No Winner

| Test | Encoding | Revert reason |
|------|----------|---------------|
| `test_decode_reverts_noWinner_allZero` | `int256(0)` | `InvalidOracleEncoding(0, 2)` |
| `test_decode_reverts_noWinner_sevenOutcome` | `int256(0)` | `InvalidOracleEncoding` |

---

### 6. Invalid: Multiple Winners

| Test | Encoding | Revert reason |
|------|----------|---------------|
| `test_decode_reverts_multipleWinners_outcome0And1` | Slots 0 and 1 both = 1 | `InvalidOracleEncoding` (duplicate winner) |
| `test_decode_reverts_multipleWinners_sevenOutcome` | Slots 3 and 6 both = 1 | `InvalidOracleEncoding` |

---

### 7. Invalid: Slot Value > 1

| Test | Encoding | Revert reason |
|------|----------|---------------|
| `test_decode_reverts_slotValueTwo` | Slot 0 = 2 | `InvalidOracleEncoding` (non-binary slot) |
| `test_decode_reverts_slotValueMaxUint32` | Slot 1 = `0xFFFFFFFF` | `InvalidOracleEncoding` |

*UMIP-183 requires each slot to be 0 or 1. Any other value is explicitly rejected.*

---

### 8. Invalid: Garbage Top Bits (Bits 224–255)

| Test | Encoding | Revert reason |
|------|----------|---------------|
| `test_decode_reverts_garbageTopBits` | `int256(1 << 224)` | `InvalidOracleEncoding` (top bits != 0) |
| `test_decode_reverts_garbageTopBitsPlusValidWinner` | Valid slot 0 + bit 224 set | `InvalidOracleEncoding` |

*These bits are the collision guard for the `UNRESOLVABLE_PRICE` sentinel (`type(int256).max` = all bits set). Any non-zero value here is rejected regardless of the slot values.*

---

### 9. Invalid: Garbage in Trailing Slots `[N, MAX_OUTCOMES)`

| Test | N | Trailing slot | Revert reason |
|------|---|---------------|---------------|
| `test_decode_reverts_trailingSlotGarbage_N2_slot2` | 2 | 2 | `InvalidOracleEncoding` |
| `test_decode_reverts_trailingSlotGarbage_N5_slot5` | 5 | 5 | `InvalidOracleEncoding` |
| `test_decode_reverts_trailingSlotGarbage_N7_noTrailing` | 7 | N/A | Slot 6 is valid; no revert |

*For a market with N outcomes, slots N through 6 are unused and must be zero. This prevents oracle implementations from accidentally setting unused label bits.*

---

### 10. Boundary — Minimum Outcome Count (N=2)

| Test | What it verifies |
|------|-----------------|
| `test_decode_minimumOutcomeCount_outcome0` | Smallest valid market (N=2), outcome 0 wins |
| `test_decode_minimumOutcomeCount_outcome1` | Smallest valid market (N=2), outcome 1 wins |

---

### 11. Fuzz Tests

| Test | Property |
|------|----------|
| `testFuzz_decode_roundTrip` | For any `(outcomeIdx, outcomeCount)` in `[0, N-1]×[2,7]`, `encodeWinner(i)` decodes back to `i` |
| `testFuzz_decode_zeroAlwaysReverts` | `decodeWinningOutcome(0, N)` always reverts for any N in `[2,7]` |
| `testFuzz_isTooEarly_onlyMinInt256` | `isTooEarly` returns true if and only if `price == type(int256).min` |
| `testFuzz_isUnresolvable_onlyMaxInt256` | `isUnresolvable` returns true if and only if `price == type(int256).max` |

---

## Helper Functions

The test file includes free-function helpers to construct specific invalid encodings:

| Helper | Description |
|--------|-------------|
| `encodeWinner(i)` | Valid: slot `i` = 1, all others = 0 |
| `encodeTwoWinners(a, b)` | Invalid: slots `a` and `b` both = 1 |
| `encodeSlotValueGtOne(i, v)` | Invalid: slot `i` = `v` (v > 1) |
| `encodeGarbageTopBits()` | Invalid: bit 224 set |
| `encodeTrailingSlotGarbage(valid, trailing)` | Invalid: valid outcome wins but trailing slot also = 1 |

---

## Note on Library Exposer

`MultiValueDecoder` uses `internal` functions. The `MultiValueDecoderExposer` contract wraps all three public-facing functions as `external` calls so Foundry can test them via normal ABI dispatch. This pattern adds zero risk to production (the exposer is test-only).
