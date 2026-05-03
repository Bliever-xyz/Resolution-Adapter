# Settlement — Architecture Reference

**Protocol:** Bliever Protocol  
**System:** `BlieverUmaAdapter` → `BlieverMarket` → `BlieverV1Pool`  
**Document Scope:** The complete settlement pipeline from oracle price delivery to final USDC claim by winners, including all accounting invariants.

---

## Executive Summary

Settlement is a three-step pipeline triggered by a single permissionless call. Every step has a strict ordering guarantee: critical settlement always completes before secondary reward handling. Pauses never block settlement or claims.

The following diagram illustrates the complete settlement sequence:

![Settlement Pipeline Diagram](https://private-us-east-1.manuscdn.com/sessionFile/RFp897btO1Cqct7PJ3MyE0/sandbox/zcR6p2bZdqlEdQRXunaVnp-images_1775923609373_na1fn_L2hvbWUvdWJ1bnR1L3NldHRsZW1lbnRfcGlwZWxpbmU.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvUkZwODk3YnRPMUNxY3Q3UEozTXlFMC9zYW5kYm94L3pjUjZwMmJaZHFsRWRRUlh1bmFWbnAtaW1hZ2VzXzE3NzU5MjM2MDkzNzNfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwzTmxkSFJzWlcxbGJuUmZjR2x3Wld4cGJtVS5wbmciLCJDb25kaXRpb24iOnsiRGF0ZUxlc3NUaGFuIjp7IkFXUzpFcG9jaFRpbWUiOjE3OTg3NjE2MDB9fX1dfQ__&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=IxixIifTsiixkRGDQQypP6QtYFOYYjBEuRlHqnM0LZD9hguETznjx0afQIwIV3xXDAWSSh~B8mPmVkD734FkLAMQ-uFhhY5QmkdrVA8OcAhkmILU~XL13AeSNfUw2veUsxl8hxDEseXo9UB-G-j712nkseHQbI0UlzIvC5-fVK0zVhun4ofDDoylDi5Rm1MP3K0zLBXkDphzNfeR5G3TtuuHeirKJtCe6SZ5rTCiFILVj-NnXepCppub4jEaK7~sefc8Zs6Kxu5IZr74NpXgp-vvfciHybY1TVpuemo01wKHO4TxCZ8I7syJXtwGOj6uAFRn~8pH~swP4XOLNpQKIg__)

---

## Step 1 — Oracle Price Retrieval and Decoding

### Entry Point
`adapter.resolve(questionId)` — permissionless, any EOA.

### Pre-checks (reverts if any fail)
- `_isInitialized(qd)`: Question must exist.
- `!qd.paused`: Per-question pause is not set.
- `!_isFlagged(qd)`: Manual resolution flag is not set.
- `!qd.resolved`: Question is not already resolved.
- `!qd.unresolvable`: Oracle didn't return `int256.max`.

### `_decodeAndResolve` — Internal Execution
1. **Settle Price:** Routes to `_questionOracle[questionId]` — the OO snapshot taken at initialization. This ensures oracle upgrades never break in-flight resolution.
2. **Sentinel Checks:** Handles `TOO_EARLY` (resets or flags) and `UNRESOLVABLE` (marks as unresolvable).
3. **Decode MULTIPLE_VALUES:** Enforces four strict rules on the `int256` encoding to ensure exactly one winner is identified.
4. **CEI Pattern:** Sets `qd.resolved = true` before calling `market.resolve(winner)`.

---

## Step 2 — Market Resolution

### Entry Point
`market.resolve(uint8 _winningOutcome)` — callable only by the `resolver` (the adapter address).

### Settlement Accounting Flow
The market computes the total payout based on trader shares, excluding the initial epsilon seed (ε) provided by the vault.

![Settlement Accounting Diagram](https://private-us-east-1.manuscdn.com/sessionFile/RFp897btO1Cqct7PJ3MyE0/sandbox/zcR6p2bZdqlEdQRXunaVnp-images_1775923609373_na1fn_L2hvbWUvdWJ1bnR1L3NldHRsZW1lbnRfYWNjb3VudGluZw.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvUkZwODk3YnRPMUNxY3Q3UEozTXlFMC9zYW5kYm94L3pjUjZwMmJaZHFsRWRRUlh1bmFWbnAtaW1hZ2VzXzE3NzU5MjM2MDkzNzNfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwzTmxkSFJzWlcxbGJuUmZZV05qYjNWdWRHbHVady5wbmciLCJDb25kaXRpb24iOnsiRGF0ZUxlc3NUaGFuIjp7IkFXUzpFcG9jaFRpbWUiOjE3OTg3NjE2MDB9fX1dfQ__&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=mLGPyjsdXr0-mHj1E--PKdSwA0u71CtLHqS6r-w28OSoqFiYjRpPmpZdKWLz0XeVCaPfbKhuHXyVfOj5gdIDEv0bHxdI5NjNb32e7B6Ic~ZWavy8BB3~8xWjDyXofGEdyj1Rp7-S6WIefx4IKtkkngFLfjL-FxCFzDbsonV00CEfk8Vl94Yf7ProuSKalGUA4N1bmQWkM3fVSln393risPiS2HoU4kxTghj1xqFc-YSvsG4yJwPB2o1cNEKBpiqvJShyf7xo2sqOKUjk8OPw1aEG7BzCwqj1dVwozfxhFY7SSAaCwO32z1~djVvIvWi0gqorzfIf30-P4l7k21CEQw__)

**Decimal Conversion:**
```
totalPayoutUsdc = _totalTraderShares[winning] / 1e12   (floor division)
```
`SHARE_TO_USDC = 1e12` bridges 18-decimal share quantities to 6-decimal USDC amounts. Floor division ensures any fractional USDC is absorbed into LP NAV, preventing locked funds.

### Interaction
`IBlieverV1Pool(pool).settleMarket(totalPayoutUsdc)`: The pool releases live vault liability, converting it into reserved funds for winners.

---

## Step 3 — Individual Winner Claims

### Entry Point
`market.claim()` — permissionless, any address, once per address. Winners must always be able to claim, even during operational pauses.

### Payout Formula
```solidity
uint256 payoutUsdc = _shares[caller][winningOutcome] / 1e12;
```
Each winning share (1e18 units) redeems for exactly 1 USDC (1e6 units).

### Dust Path
If `shares18 < 1e12`, the payout is 0. The contract sets `_claimed[caller] = true` and emits `DustForfeited`. The dust is absorbed into LP NAV, as it was already excluded from the pool's settlement amount.

---

## Settlement Accounting Invariants

### Vault Safety (LS-LMSR Proposition 4.9)
```
vault profit = riskBudget - totalPayoutUsdc >= 0
```
The LS-LMSR cost function guarantees that the maximum possible payout is bounded by the pool's `maxRiskPerMarket`. The AMM's spread ensures the vault is always profitable or break-even.

### Pool Liability Lifecycle
1. **Registration:** `pool.setMarketLiability(riskBudget)`
2. **Each Trade:** Updates live liability via `collectTradeCost` or `distributeRefund`.
3. **Settlement:** `pool.settleMarket(totalPayoutUsdc)` (converts live → reserved).
4. **Claims:** `pool.claimWinnings` (depletes reserved).
5. **Post-claims:** `pool.revokeMarketRole` (removes market from active set).

---

## MULTIPLE_VALUES Encoding Reference

For an N-outcome market, the oracle returns `int256 encodedPrice` where:
- **Bits 0–31:** Outcome 0 value (uint32)
- **Bits 32–63:** Outcome 1 value (uint32)
- **...**
- **Bits 224–255:** Always 0 (collision guard)

Sentinel values checked before decoding:
- `type(int256).min` → `TOO_EARLY`
- `type(int256).max` → `UNRESOLVABLE`

---

## Key Events Emitted During Settlement

| Event | Emitter | When |
| :--- | :--- | :--- |
| `QuestionResolved` | Adapter | After `market.resolve()` succeeds |
| `MarketResolved` | Market | Inside `resolve()` |
| `WinningsClaimed` | Market | Each successful `claim()` |
| `DustForfeited` | Market | When winning shares < 1e12 |
| `RefundFailed` | Adapter | If best-effort reward refund fails |
| `MarketExpired` | Market | On `expireUnresolved()` |
