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
| 1 | `uint256(encodedPrice) & TOP_BITS_MASK == 0` | `InvalidOracleEncoding(encodedPrice, outcomeCount)` |
| 2 | All slots `[outcomeCount, MAX_OUTCOMES)` equal 0 | `InvalidOracleEncoding(encodedPrice, outcomeCount)` |
| 3 | Exactly one slot in `[0, outcomeCount)` equals 1 | `InvalidOracleEncoding(encodedPrice, outcomeCount)` |
| 4 | No slot in `[0, outcomeCount)` exceeds 1 | `InvalidOracleEncoding(encodedPrice, outcomeCount)` |

---

## 4. Internal Helper

### `_slotAt(uint256 raw, uint256 index) → uint32`
```solidity
return uint32((raw >> (32 * index)) & UINT32_MASK);
```
Extracts the `uint32` value at the given 0-based index from a packed `uint256`.

---

## 5. Error

| Error | Signature | Thrown by | Meaning |
|---|---|---|---|
| `InvalidOracleEncoding` | `InvalidOracleEncoding(int256 price, uint8 outcomeCount)` | `decodeWinningOutcome` | Price does not satisfy the single-winner contract |

The error carries the exact `price` that failed validation and the `outcomeCount` that was in scope. Both parameters are ABI-encoded into the reverted call's returndata — there is no on-chain storage write and no gas cost beyond that of a bare custom error. Off-chain consumers (monitoring bots, Foundry test output, block explorers) can decode the revert data directly without replaying the transaction.

---

## 6. Testing the Decoder (Quick Reference)

Encoding test prices does not require a library function. Construct them directly from the UMIP-183 bit layout:

```solidity
// Encode: slot i = value  →  int256(uint256(value) << (32 * i))

// 3-outcome market, outcome 1 wins: slot[1] = 1
int256 price = int256(uint256(1) << 32); // 4294967296

uint8 winner = MultiValueDecoder.decodeWinningOutcome(price, 3);
// winner == 1 ✓

// Sentinel checks
assert(MultiValueDecoder.isTooEarly(type(int256).min));
assert(MultiValueDecoder.isUnresolvable(type(int256).max));

// Should revert: two winners (slot[0] = 1, slot[1] = 1)
int256 twoWinners = int256((uint256(1) << 0) | (uint256(1) << 32));
// decodeWinningOutcome(twoWinners, 3)
// → InvalidOracleEncoding(twoWinners, 3)

// Should revert: no winner
int256 noWinner = 0;
// decodeWinningOutcome(noWinner, 3)
// → InvalidOracleEncoding(noWinner, 3)

// Should revert: top bits set (crafted attack price)
int256 topBits = int256(uint256(1) << 224);
// decodeWinningOutcome(topBits, 3)
// → InvalidOracleEncoding(topBits, 3)

// Should revert: slot value > 1 (slot[0] = 2)
int256 badSlot = int256(uint256(2));
// decodeWinningOutcome(badSlot, 3)
// → InvalidOracleEncoding(badSlot, 3)
```

The error payload is directly readable in Foundry test output:
```
[FAIL] InvalidOracleEncoding(price: 4294967298, outcomeCount: 3)
```