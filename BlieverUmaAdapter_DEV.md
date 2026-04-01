# BlieverUmaAdapter — Developer Implementation Reference

> **Document type:** Technical implementation explanation.  
> **Audience:** Developers who want to analyze, improve, debug, test, or integrate the contract.  
> **Companion:** `BlieverUmaAdapter.md` covers the theoretical architecture and rationale.

---

## Contract Map

```
contracts/src/
├── BlieverUmaAdapter.sol          ← Main adapter (UUPS proxy)
├── libraries/
│   └── MultiValueDecoder.sol      ← UMIP-183 encode / decode library (pure)
├── mixins/
│   └── BulletinBoard.sol          ← On-chain update registry (Polymarket, unchanged)
└── interfaces/
    ├── IBlieverUmaAdapter.sol     ← Public interface + QuestionData struct
    ├── IBlieverMarket.sol         ← Minimal market interface (resolve, questionId, outcomeCount)
    ├── IOptimisticOracleV2.sol    ← Minimal UMA OO interface
    ├── IOptimisticRequester.sol   ← UMA callback interface
    └── IBulletinBoard.sol         ← BulletinBoard interface + AncillaryDataUpdate struct
```

---

## 1. `QuestionData` Struct — Storage Layout

Defined in `IBlieverUmaAdapter.sol`. **Never reorder fields** — existing proxy storage would be corrupted.

```
Slot 0 (31 bytes used):
  address market            (20 bytes) — BlieverMarket clone address
  uint40  requestTimestamp  (5 bytes)  — updated on each OO reset
  uint40  manualResolveAt   (5 bytes)  — 0 unless flagged
  uint8   outcomeCount      (1 byte)   — cached from market.outcomeCount()

Slot 1 (25 bytes used):
  bool    resolved          (1 byte)
  bool    paused            (1 byte)
  bool    reset             (1 byte)   — true after first dispute
  bool    refund            (1 byte)   — true when reward sits on adapter
  bool    unresolvable      (1 byte)   — true when OO returned int256.max
  address rewardToken       (20 bytes)

Slot 2 (20 bytes used):
  address creator           (20 bytes)

Slots 3–5 (32 bytes each):
  uint256 reward
  uint256 proposalBond
  uint256 liveness

Dynamic slot:
  bytes   ancillaryData     — keccak256(ancillaryData) == questionId
```

The `questions` mapping in `BlieverUmaAdapter` starts at storage slot determined by `AccessControlUpgradeable` + `PausableUpgradeable` + `UUPSUpgradeable` + `BulletinBoard` inheritance. The key point: `BulletinBoard.updates` occupies one mapping slot, and `optimisticOracle` + `questions` follow in declaration order.

---

## 2. `MultiValueDecoder` Library

**File:** `contracts/src/libraries/MultiValueDecoder.sol`  
**Type:** Pure library — no state, no external calls.

### Key Constants

| Constant | Value | Purpose |
|---|---|---|
| `MAX_OUTCOMES` | `7` | V1 hard cap (UMIP-183 maximum) |
| `TOO_EARLY_PRICE` | `type(int256).min` | Sentinel: event-based expiry before game start |
| `UNRESOLVABLE_PRICE` | `type(int256).max` | Sentinel: event canceled / ancillary data invalid |
| `UINT32_MASK` | `0xFFFFFFFF` | Isolates one 32-bit slot |
| `TOP_BITS_MASK` | `0xFFFFFFFF << 224` | Guards bits 224–255 (must be zero per UMIP-183) |

### `decodeWinningOutcome(int256 encodedPrice, uint8 outcomeCount) → uint8`

**Four sequential validation rules** (all must pass, else `InvalidOracleEncoding`):

1. **Top 32 bits zero**: `raw & TOP_BITS_MASK == 0`.  
   Guards against crafted prices where the top bits are set but the price is not `type(int256).max`.

2. **Trailing slots zero**: For `i ∈ [outcomeCount, MAX_OUTCOMES)`, `_slotAt(raw, i) == 0`.  
   Prevents garbage data in unused UMIP-183 label positions.

3–4. **Exactly one winner**: Iterates `[0, outcomeCount)`. Exactly one slot must equal `1`; all others must equal `0`. Any slot value `> 1` or a second slot with value `1` reverts.

**Critical order of checks in `_decodeAndResolve`:**
```solidity
if (MultiValueDecoder.isTooEarly(encodedPrice))   → _resetQuestion
if (MultiValueDecoder.isUnresolvable(encodedPrice)) → mark unresolvable
// Only THEN call decodeWinningOutcome — sentinels are excluded
```
Per UMIP-183: "to prevent misinterpretation prices should first be checked against the special values before decoding."

---

## 3. `BulletinBoard` Mixin

**File:** `contracts/src/mixins/BulletinBoard.sol`  
**Source:** Polymarket's `uma-ctf-adapter` — core logic unchanged, pragma updated 0.8.15 → 0.8.31.

Storage: one mapping `updates[keccak256(abi.encode(questionID, poster))] → AncillaryDataUpdate[]`.

`postUpdate(questionID, update)` — permissionless. Any address may post.  
`getUpdates(questionID, owner)` — returns all updates from a specific poster.  
`getLatestUpdate(questionID, owner)` — returns the most recent update or a zero struct.

---

## 4. `BlieverUmaAdapter` — Function-by-Function

### 4.1 `initialize(optimisticOracle, admin, factory, emergency)`

- Calls `__AccessControl_init()`, `__Pausable_init()`, `__UUPSUpgradeable_init()`.
- Grants `DEFAULT_ADMIN_ROLE`, `FACTORY_ROLE`, `EMERGENCY_ROLE` to the provided addresses.
- Sets `optimisticOracle` storage slot.
- `_disableInitializers()` in the constructor prevents the implementation contract from being initialized directly.

---

### 4.2 `initializeQuestion(questionId, market, ancillaryData, rewardToken, reward, proposalBond, liveness)`

**Access:** `FACTORY_ROLE` + `whenNotPaused`  
**Reentrancy:** `nonReentrant`

**Validation sequence:**
```
market != address(0) && rewardToken != address(0)
ancillaryData.length ∈ [1, MAX_ANCILLARY_DATA]
keccak256(ancillaryData) == questionId           ← off-chain / on-chain consistency
market.questionId() == questionId                ← correct market bound to adapter
!_isInitialized(questions[questionId])           ← no duplicate registration
market.outcomeCount() ∈ [2, MAX_OUTCOMES]        ← V1 cap
```

**Then:**
1. Writes full `QuestionData` to `questions[questionId]` (single SSTORE batch).
2. Calls `_requestPrice(msg.sender, requestTimestamp, …)` which:
   - Pulls `reward` from factory → adapter via `safeTransferFrom` if `reward > 0`.
   - Approves OO for `type(uint256).max` if current allowance < reward.
   - Calls `OO.requestPrice(MULTIPLE_VALUES, …)`.
   - Calls `OO.setEventBased(…)`.
   - Calls `OO.setCallbacks(…, disputeCallback=true, …)`.
   - Calls `OO.setBond(…)` if `proposalBond > 0`.
   - Calls `OO.setCustomLiveness(…)` if `liveness > 0`.

---

### 4.3 `resolve(questionId)` — Permissionless

**Access:** public + `whenNotPaused`  
**Reentrancy:** `nonReentrant`

**Guard checks (fast-revert order):**
```
_isInitialized(qd)  → NotInitialized
qd.paused           → QuestionIsPaused
qd.resolved         → AlreadyResolved
qd.unresolvable     → Unresolvable
_hasPrice(qd)       → PriceNotAvailable
```

Delegates to `_decodeAndResolve(questionId, qd)`.

**`_decodeAndResolve` internals:**
```
int256 encodedPrice = OO.settleAndGetPrice(MULTIPLE_VALUES, qd.requestTimestamp, qd.ancillaryData)

if isTooEarly(encodedPrice):
    _resetQuestion(address(this), questionId, true, qd)
    return

if isUnresolvable(encodedPrice):
    qd.unresolvable = true
    emit QuestionUnresolvable
    return

uint8 winner = decodeWinningOutcome(encodedPrice, qd.outcomeCount)

// ── CEI: Effects ────────────────────────────────────────────────
qd.resolved = true
if qd.refund && qd.reward > 0: _refund(qd)          // ERC-20 transfer

// ── Interaction ─────────────────────────────────────────────────
IBlieverMarket(qd.market).resolve(winner)
emit QuestionResolved
```

**Note on OO.settleAndGetPrice timing:** This call happens *before* the CEI effects. This is acceptable because:
1. `ReentrancyGuardTransient` blocks any re-entrant call from the OO.
2. `settleAndGetPrice` is required to get the price — there is no alternative.
3. The OO is a trusted, audited UMA contract.

---

### 4.4 `priceDisputed(identifier, timestamp, ancillaryData, refund)` — OO Callback

**Access:** `onlyOptimisticOracle`

```
bytes32 questionId = keccak256(ancillaryData)
QuestionData storage qd = questions[questionId]

if qd.resolved:                    // Admin already resolved manually
    transfer qd.reward to qd.creator
    return

if qd.reset:                       // Second dispute → DVM
    qd.refund = true
    return

// First dispute → reset
_resetQuestion(address(this), questionId, false, qd)
```

**Why `address(this)` pays for the reset:** On a first dispute, the reward already on the adapter is reused for the new OO request (the adapter is the `requestor`). On a second dispute (`qd.refund = true`), the reward is returned to the creator upon final resolution.

---

### 4.5 `flag(questionId)` + `unflag(questionId)` + `resolveManually(questionId, winningOutcome)`

**Access:** `EMERGENCY_ROLE` + `nonReentrant`

**`flag`:**
```
check: initialized, not flagged, not resolved
qd.manualResolveAt = block.timestamp + SAFETY_PERIOD
qd.paused = true
```

**`unflag`:**
```
check: initialized, flagged, not resolved, block.timestamp < manualResolveAt
qd.manualResolveAt = 0
qd.paused = false
```

**`resolveManually`:**
```
check: initialized, flagged, not resolved, block.timestamp >= manualResolveAt
check: winningOutcome < qd.outcomeCount

// CEI
qd.resolved = true
if qd.refund && qd.reward > 0: _refund(qd)   // ERC-20 transfer
IBlieverMarket(qd.market).resolve(winningOutcome)
```

---

### 4.6 `reset(questionId)`

**Access:** `EMERGENCY_ROLE` + `nonReentrant`  
**Use case:** `priceDisputed` callback failed to fire (OO-side bug). Admin manually issues a fresh OO request.

```
check: initialized, not resolved
if qd.refund && qd.reward > 0: _refund(qd)
_resetQuestion(msg.sender, questionId, true, qd)
```

`msg.sender` (admin) pays for the new OO request reward, not `address(this)`.

---

### 4.7 `_resetQuestion(requestor, questionId, resetRefund, qd)`

```
qd.requestTimestamp = uint40(block.timestamp)   // New OO anchor timestamp
qd.reset = true
if resetRefund: qd.refund = false
_requestPrice(requestor, newTimestamp, qd.ancillaryData, …)
emit QuestionReset
```

**Critical:** `qd.requestTimestamp` must be updated before `_requestPrice()` is called. The OO identifies the request by `(requester, identifier, timestamp, ancillaryData)` — the new timestamp creates a distinct request key, not an update to the old one.

---

### 4.8 `updateOptimisticOracle(newOracle)`

**Access:** `DEFAULT_ADMIN_ROLE` + `nonReentrant`

Updates `optimisticOracle` storage slot. The `onlyOptimisticOracle` modifier reads this slot live, so immediately after the update:
- New `priceDisputed` callbacks are accepted from `newOracle` only.
- Calls to `OO.settleAndGetPrice` in `resolve()` use `newOracle`.

**Historical requests:** Questions already in the system store their `requestTimestamp` as the OO request key. If they need their historical request settled, they call the old OO's `settleAndGetPrice` via the *current adapter's `optimisticOracle` pointer* — which now points to `newOracle`. This means the caller needs to account for this if historical requests were made on the old OO. **Recommendation:** Do not update the OO address while there are unresolved questions. Use the upgrade to migrate to a new OO for new questions only, after all live questions have settled.

---

## 5. `_hasPrice` and `_ready`

```solidity
function _hasPrice(QuestionData storage qd) internal view returns (bool) {
    return optimisticOracle.hasPrice(
        address(this),
        MULTIPLE_VALUES_IDENTIFIER,
        qd.requestTimestamp,    // Uses the LATEST requestTimestamp (updated on reset)
        qd.ancillaryData
    );
}

function _ready(QuestionData storage qd) internal view returns (bool) {
    return _isInitialized(qd)
        && !qd.paused
        && !qd.resolved
        && !qd.unresolvable
        && _hasPrice(qd);
}
```

`_hasPrice` uses `qd.requestTimestamp` — after a `_resetQuestion`, the new timestamp is used, so `hasPrice` checks the reset request, not the original.

---

## 6. Role Enforcement — Modifier Summary

| Modifier | Enforced by | Applied on |
|---|---|---|
| `onlyRole(FACTORY_ROLE)` | OZ AccessControl | `initializeQuestion` |
| `onlyRole(EMERGENCY_ROLE)` | OZ AccessControl | `flag`, `unflag`, `resolveManually`, `reset`, `pauseQuestion`, `unpauseQuestion` |
| `onlyRole(DEFAULT_ADMIN_ROLE)` | OZ AccessControl | `pause`, `unpause`, `updateOptimisticOracle` |
| `onlyOptimisticOracle` | `msg.sender == address(optimisticOracle)` | `priceDisputed` callback |
| `whenNotPaused` | OZ Pausable | `initializeQuestion`, `resolve` |
| `nonReentrant` | OZ ReentrancyGuardTransient (EIP-1153) | All state-mutating externals |

---

## 7. UUPS Upgrade Notes

`_authorizeUpgrade(newImplementation)` is gated by `DEFAULT_ADMIN_ROLE`.

**Storage invariants across upgrades:**
- `QuestionData` struct field order must not change.
- `BulletinBoard.updates` mapping slot must not change (inherited first in the linearization after Initializable).
- `optimisticOracle` slot must not change.
- `questions` mapping slot must not change.
- Adding new storage variables is safe only by appending (do NOT insert between existing variables or inherited contracts).

**Safe upgrade pattern:** Use OpenZeppelin's storage gap pattern (`uint256[50] private __gap`) in any new base contracts added in V2 to reserve future slots.

---

## 8. Integration Notes for the MarketFactory

The factory's `deployMarket()` transaction must:

1. Deploy the EIP-1167 clone with `resolver = address(adapter)`.
2. Call `market.initialize(…, _resolver = address(adapter), …)`.
3. Call `adapter.initializeQuestion(questionId, market, ancillaryData, rewardToken, reward, bond, liveness)`.
   - Factory must pre-approve `adapter` as spender for `reward` amount of `rewardToken`.

The `questionId` passed to both `market.initialize` and `adapter.initializeQuestion` **must** equal `keccak256(ancillaryData)`. This is validated on-chain in `initializeQuestion`.

### Resolving a market (post-event):

```solidity
// Off-chain: wait for adapter.ready(questionId) == true
// Then (any EOA):
adapter.resolve(questionId);
// Results in market.resolved() == true and pool.settleMarket() called internally
```

### Handling unresolvable markets:

```solidity
// If adapter.getQuestion(questionId).unresolvable == true
// AND block.timestamp > market.resolutionDeadline():
factory.callExpireUnresolved(marketAddress);
// market.expireUnresolved() calls pool.settleMarket(0)
```

---

## 9. Key Error Conditions and Debug Guide

| Error | Cause | Fix |
|---|---|---|
| `QuestionIdMismatch` | `keccak256(ancillaryData) != questionId` in `initializeQuestion` | Recompute questionId off-chain from exact ancillaryData bytes |
| `QuestionIdMismatch` | `market.questionId() != questionId` | Market was initialized with wrong questionId; redeploy |
| `AlreadyInitialized` | `initializeQuestion` called twice for same questionId | Check `isInitialized(questionId)` before calling |
| `InvalidOutcomeCount` | `market.outcomeCount() > 7` | V1 cap; use outcomeCount ∈ [2, 7] |
| `PriceNotAvailable` | `resolve()` called before liveness window closed | Check `ready(questionId)` first; poll until true |
| `NotOptimisticOracle` | `priceDisputed` called by wrong address | OO address mismatch; check `optimisticOracle()` on adapter |
| `InvalidOracleEncoding` | Oracle returned a price with multiple winners, a value > 1, or non-zero trailing bits | Review proposer bot encoding; ensure it uses `encodeValues([0,..,1,..,0])` |
| `Unresolvable` | `resolve()` called after event was canceled (oracle returned int256.max) | Check `getQuestion(id).unresolvable`; trigger `factory.expireUnresolved()` |
| `SafetyPeriodNotPassed` | `resolveManually` called too early | Wait until `block.timestamp >= getQuestion(id).manualResolveAt` |
| `SafetyPeriodPassed` | `unflag()` called after the safety window closed | Cannot unflag; must proceed with `resolveManually` |

---

## 10. Gas Notes

- `initializeQuestion` is the most gas-intensive call: writes 6+ storage slots + 3–5 OO external calls. Expected ~250k–300k gas on Base.
- `resolve()` (fast path, no dispute): ~80k–120k gas. `settleAndGetPrice` + `decodeWinningOutcome` (pure math) + `market.resolve()` (1 SSTORE resolved + pool.settleMarket).
- `priceDisputed` callback: ~60k gas. Two SSTOREs + one new OO request.
- `postUpdate` (BulletinBoard): ~25k gas (one SSTORE for the new array element).
- `ReentrancyGuardTransient` (EIP-1153): ~0 persistent gas overhead vs ~20k for the classic guard. Requires EIP-1153 support (Base supports this from genesis).
