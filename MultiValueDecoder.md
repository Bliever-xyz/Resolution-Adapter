# MultiValueDecoder — Architecture & Developer Reference

> **Document type:** Combined architecture + implementation reference (library is small enough to warrant a single doc).  
> **Audience:** Developers integrating, testing, or auditing the UMIP-183 decode path.

---

## 1. Purpose

`MultiValueDecoder` is a pure Solidity library implementing the UMIP-183 `MULTIPLE_VALUES` encoding specification. It is used exclusively by `BlieverUmaAdapter._decodeAndResolve()` to translate the oracle's settled `int256` into the `uint8 winningOutcome` that `BlieverMarket.resolve()` expects.

**It has zero state, zero external calls, and zero side-effects.**

---

## 2. UMIP-183 Encoding Contract

The UMA oracle packs up to 7 × `uint32` values into a single `int256`:

```
int256 layout:
┌──────────────┬────────────┬────────────┬─────┬────────────┐
│ bits 224–255 │ bits 192–  │ bits 160–  │ … │ bits  0–31  │
│   always 0   │   223      │   191      │   │             │
│   (unused)   │  outcome6  │  outcome5  │   │  outcome0   │
└──────────────┴────────────┴────────────┴─────┴────────────┘
```

**Value at index `i`:** `uint32( (uint256(encodedPrice) >> (32 * i)) & 0xFFFFFFFF )`

### Special Sentinels (MUST be checked before decoding)

| Value | Meaning |
|---|---|
| `type(int256).min` | Too early — proposal was submitted before the event anchor timestamp |
| `type(int256).max` | Unresolvable — event canceled, ancillary data invalid, or > 7 labels |

### Bliever Single-Winner Contract

For a BlieverMarket with N outcomes, the oracle must return a price where:
- Exactly **one** slot in `[0, N)` equals `1` (the winning outcome).
- All other slots in `[0, N)` equal `0`.
- All slots in `[N, 7)` equal `0` (unused labels).
- Bits 224–255 equal `0`.

Example for a 3-outcome market where outcome 1 wins:
```
encodedPrice = 1 << 32 = 4294967296
slot[0] = 0, slot[1] = 1, slot[2] = 0 ✓
```

---

## 3. Functions

### `isTooEarly(int256 price) → bool`
Returns `price == type(int256).min`. Call this **first** before decoding.

### `isUnresolvable(int256 price) → bool`
Returns `price == type(int256).max`. Call this **second** before decoding.

### `decodeWinningOutcome(int256 encodedPrice, uint8 outcomeCount) → uint8 winningOutcome`

Four validation rules in order:

| Step | Check | Error on failure |
|---|---|---|
| 1 | `uint256(encodedPrice) & TOP_BITS_MASK == 0` | `InvalidOracleEncoding` |
| 2 | All slots `[outcomeCount, MAX_OUTCOMES)` equal 0 | `InvalidOracleEncoding` |
| 3 | Exactly one slot in `[0, outcomeCount)` equals 1 | `InvalidOracleEncoding` |
| 4 | No slot in `[0, outcomeCount)` exceeds 1 | `InvalidOracleEncoding` |

### `encodeValues(uint32[] memory values) → int256 encodedPrice`
Utility function for off-chain tooling and tests. Packs an array of uint32 values into the UMIP-183 int256. `values.length` must be ≤ `MAX_OUTCOMES`. **Not used in the production resolution hot path.**

---

## 4. Internal Helper

### `_slotAt(uint256 raw, uint256 index) → uint32`
```solidity
return uint32((raw >> (32 * index)) & UINT32_MASK);
```
Extracts the `uint32` value at the given 0-based index from a packed `uint256`.

---

## 5. Error

| Error | Thrown by | Meaning |
|---|---|---|
| `InvalidOracleEncoding` | `decodeWinningOutcome` | Price does not satisfy the single-winner contract |

---

## 6. Testing the Decoder (Quick Reference)

```solidity
// Setup: 3-outcome market, outcome 1 wins
uint32[] memory vals = new uint32[](3);
vals[1] = 1;
int256 price = MultiValueDecoder.encodeValues(vals);
// price == 4294967296

uint8 winner = MultiValueDecoder.decodeWinningOutcome(price, 3);
// winner == 1 ✓

// Sentinel checks
assert(MultiValueDecoder.isTooEarly(type(int256).min));
assert(MultiValueDecoder.isUnresolvable(type(int256).max));

// Should revert: two winners
vals[0] = 1;
MultiValueDecoder.encodeValues(vals);      // price has both slot 0 and slot 1 = 1
MultiValueDecoder.decodeWinningOutcome(price, 3); // → InvalidOracleEncoding

// Should revert: no winner
int256 noWinner = 0;
MultiValueDecoder.decodeWinningOutcome(noWinner, 3); // → InvalidOracleEncoding

// Should revert: top bits set (crafted attack price)
int256 topBits = int256(uint256(1) << 224);
MultiValueDecoder.decodeWinningOutcome(topBits, 3); // → InvalidOracleEncoding
```
