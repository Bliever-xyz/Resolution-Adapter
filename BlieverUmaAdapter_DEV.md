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
│   ├── MultiValueDecoder.sol      ← UMIP-183 encode / decode library (pure)
│   └── AncillaryDataLib.sol       ← Initializer-append + gas-optimised address encoder (pure)
├── mixins/
│   └── BulletinBoard.sol          ← On-chain update registry (Polymarket, unchanged)
└── interfaces/
    ├── IBlieverUmaAdapter.sol     ← Public interface + QuestionData struct  ⚠ see note
    ├── IBlieverMarket.sol         ← Minimal market interface (resolve, questionId, outcomeCount)
    ├── IOptimisticOracleV2.sol    ← Minimal UMA OO interface
    ├── IOptimisticRequester.sol   ← UMA callback interface
    └── IBulletinBoard.sol         ← BulletinBoard interface + AncillaryDataUpdate struct
```

> **⚠ `IBlieverUmaAdapter.sol` requires an update.**  
> `QuestionEscalatedToDVM` and `RefundFailed` are declared in `BlieverUmaAdapter.sol` directly (not in the interface) because they were introduced after the initial interface cut. They emit correctly on-chain in all cases. However, until they are added to `IBlieverUmaAdapter.sol`, external consumers that ABI-decode from the interface (SDKs, subgraphs, ethers.js `Contract` instances typed against the interface) will not see these events. Add both declarations to `IBlieverUmaAdapter.sol`:
>
> ```solidity
> event QuestionEscalatedToDVM(bytes32 indexed questionId);
> event RefundFailed(bytes32 indexed questionId, address indexed creator, uint256 amount, address indexed token);
> ```

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
  bytes   ancillaryData     — fullAncillaryData: raw JSON ++ ",initializer:" ++ hex(factory)
                              keccak256(ancillaryData) == questionId
```

The `questions` mapping in `BlieverUmaAdapter` starts at storage slot determined by `AccessControlUpgradeable` + `PausableUpgradeable` + `UUPSUpgradeable` + `BulletinBoard` inheritance. The key point: `BulletinBoard.updates` occupies one mapping slot, and `optimisticOracle`, `questions`, `_knownOracles`, and `_questionOracle` follow in declaration order.

### Additional Storage — Oracle Tracking

```
mapping(address => bool)                 _knownOracles     (private)
mapping(bytes32 => IOptimisticOracleV2)  _questionOracle   (private)
```

`_knownOracles` records every OO address ever assigned to this adapter. Used by the `onlyOptimisticOracle` modifier to accept `priceDisputed` callbacks from any historically valid oracle, not only the current one.

`_questionOracle` records the OO instance active at the time each question was initialized or last reset. All `hasPrice` and `settleAndGetPrice` calls for a given question route to this address, isolating resolution from any subsequent `updateOptimisticOracle` call.

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
// using MultiValueDecoder for int256 — dot notation used throughout
if (encodedPrice.isTooEarly())      → branch on reward (reset or flag)
if (encodedPrice.isUnresolvable())  → mark unresolvable
uint8 w = encodedPrice.decodeWinningOutcome(outcomeCount)
// Sentinels are excluded before decoding per UMIP-183.
```
Per UMIP-183: "to prevent misinterpretation prices should first be checked against the special values before decoding."

---

## 3. `AncillaryDataLib` Library

**File:** `contracts/src/libraries/AncillaryDataLib.sol`  
**Source:** UMA Protocol / Polymarket `uma-ctf-adapter` — pragma updated to 0.8.31.  
**Type:** Pure library — no state, no external calls.

### Purpose

Appends a canonical `,initializer:<hex-address>` suffix to raw ancillary data bytes before they are hashed and submitted to the UMA Optimistic Oracle. This attributes each market to the factory address on the UMA DVM voter UI, allowing voters to distinguish protocol-sanctioned markets from spam.

### `_appendAncillaryData(address initializer, bytes memory ancillaryData) → bytes memory`

Returns `abi.encodePacked(ancillaryData, ",initializer:", _toUtf8BytesAddress(initializer))`.

The returned bytes are what the adapter stores in `qd.ancillaryData` and submits to the OO. All downstream operations — `_requestPrice`, `_resetQuestion`, `_hasPrice`, `_decodeAndResolve`, and the `priceDisputed` callback — operate on this full form.

### `_toUtf8BytesAddress(address addr) → bytes memory`

Converts a 20-byte address to a 40-character lower-case hex ASCII string (no `0x` prefix), producing 40 bytes total. Delegates to `_toUtf8Bytes32Bottom` for bit-manipulation hex encoding.

### `_toUtf8Bytes32Bottom(bytes32 bytesIn) → bytes32` (private)

Gas-optimised nibble interleave + hex encode operating entirely in `uint256` arithmetic inside an `unchecked` block. Sourced from the UMA Protocol repository and used verbatim by Polymarket. Encoding cost is approximately 500 gas per byte versus ~2 500 gas for a naive string-conversion loop.

### Suffix length

| Component | Bytes |
|---|---|
| `,initializer:` prefix | 13 |
| Lower-case hex address (no `0x`) | 40 |
| **Total suffix** | **53** |

This is the value of `INITIALIZER_SUFFIX_LENGTH` in the adapter.

---

## 4. `BulletinBoard` Mixin

**File:** `contracts/src/mixins/BulletinBoard.sol`  
**Source:** Polymarket's `uma-ctf-adapter` — core logic unchanged, pragma updated 0.8.15 → 0.8.31.

Storage: one mapping `updates[keccak256(abi.encode(questionID, poster))] → AncillaryDataUpdate[]`.

`postUpdate(questionID, update)` — permissionless. Any address may post.  
`getUpdates(questionID, owner)` — returns all updates from a specific poster.  
`getLatestUpdate(questionID, owner)` — returns the most recent update or a zero struct.

---

## 5. `BlieverUmaAdapter` — Function-by-Function

### 5.1 `initialize(optimisticOracle, admin, factory, emergency)`

- Calls `__AccessControl_init()`, `__Pausable_init()`, `__UUPSUpgradeable_init()`.
- Grants `DEFAULT_ADMIN_ROLE`, `FACTORY_ROLE`, `EMERGENCY_ROLE` to the provided addresses.
- Sets `optimisticOracle` storage slot.
- Registers the initial oracle in `_knownOracles[_optimisticOracle] = true`.
- `_disableInitializers()` in the constructor prevents the implementation contract from being initialized directly.

---

### 5.2 `initializeQuestion(questionId, market, ancillaryData, rewardToken, reward, proposalBond, liveness)`

**Access:** `FACTORY_ROLE` + `whenNotPaused`  
**Reentrancy:** `nonReentrant`

**Validation sequence:**
```
market != address(0) && rewardToken != address(0)
ancillaryData.length ∈ [1, MAX_ANCILLARY_DATA − INITIALIZER_SUFFIX_LENGTH]
                                                 ← raw bytes must leave room for 53-byte suffix
fullAncillaryData = AncillaryDataLib._appendAncillaryData(msg.sender, ancillaryData)
keccak256(fullAncillaryData) == questionId       ← off-chain / on-chain consistency
market.questionId() == questionId                ← correct market bound to adapter
!_isInitialized(questions[questionId])           ← no duplicate registration
market.outcomeCount() ∈ [2, MAX_OUTCOMES]        ← V1 cap
```

**Then:**
1. Computes `fullAncillaryData = AncillaryDataLib._appendAncillaryData(msg.sender, ancillaryData)`,
   appending `,initializer:<lower-case-hex-address>` to attribute the market to its factory on the UMA DVM voter UI.
2. Writes full `QuestionData` to `questions[questionId]` (single SSTORE batch), storing `fullAncillaryData`.
3. **Snapshots `_questionOracle[questionId] = optimisticOracle`** — pins this question to the current OO for all future `hasPrice` and `settleAndGetPrice` routing.
4. Calls `_requestPrice(msg.sender, requestTimestamp, fullAncillaryData, …)` which:
   - Pulls `reward` from factory → adapter via `safeTransferFrom` if `reward > 0`.
   - Approves OO for `type(uint256).max` if current allowance < reward.
   - Calls `OO.requestPrice(MULTIPLE_VALUES, …)`.
   - Calls `OO.setEventBased(…)`.
   - Calls `OO.setCallbacks(…, disputeCallback=true, …)`.
   - Calls `OO.setBond(…)` if `proposalBond > 0`.
   - Calls `OO.setCustomLiveness(…)` if `liveness > 0`.

---

### 5.3 `resolve(questionId)` — Permissionless

**Access:** public + `whenNotPaused`  
**Reentrancy:** `nonReentrant`

**Guard checks (fast-revert order):**
```
_isInitialized(qd)   → NotInitialized
qd.paused            → QuestionIsPaused
_isFlagged(qd)       → Flagged          ← independent of qd.paused
qd.resolved          → AlreadyResolved
qd.unresolvable      → Unresolvable
```

`_isFlagged` is checked independently of `qd.paused`. A question can be flagged without being paused, or paused without being flagged, and both checks gate the resolution path independently.

`_hasPrice` is **not** called in `resolve()`. `settleAndGetPrice()` inside `_decodeAndResolve()` naturally reverts with the OO's own error if the liveness window has not closed. This eliminates a redundant external view call to the OO on every resolution. `ready()` remains the canonical off-chain / indexer signal for price availability.

Delegates to `_decodeAndResolve(questionId, qd)`.

**`_decodeAndResolve` internals:**
```
int256 encodedPrice = _questionOracle[questionId].settleAndGetPrice(
    MULTIPLE_VALUES, qd.requestTimestamp, qd.ancillaryData
)
// Routes to the OO instance that holds the actual settled request,
// not the current global optimisticOracle. Safe across oracle upgrades.

if encodedPrice.isTooEarly():          // dot notation — using MultiValueDecoder for int256
    if qd.reward > 0:
        // OO consumed the reward on TOO_EARLY — adapter balance is 0.
        // Self-funded reset would revert on safeTransferFrom(this, OO, reward).
        qd.manualResolveAt = block.timestamp + SAFETY_PERIOD
        emit QuestionFlagged
    else:
        // reward == 0: no token at risk. Auto-reset is safe.
        _resetQuestion(address(this), questionId, true, qd)
    return

if encodedPrice.isUnresolvable():      // dot notation
    qd.unresolvable = true
    emit QuestionUnresolvable
    return

uint8 winner = encodedPrice.decodeWinningOutcome(qd.outcomeCount)   // dot notation

// ── CEI: Effects ────────────────────────────────────────────────
qd.resolved = true

// ── Critical interaction — must never be blocked ────────────────
IBlieverMarket(qd.market).resolve(winner)
// market.resolve executes FIRST so a stuck reward token transfer
// (e.g. USDC blacklist on creator) cannot prevent market settlement.

// ── Secondary interaction — non-reverting best-effort refund ────
if qd.refund && qd.reward > 0: _bestEffortRefund(questionId, qd)
// Uses try/catch internally. On failure emits RefundFailed; market
// settlement above is unaffected.

emit QuestionResolved
```

**Note on OO.settleAndGetPrice timing:** This call happens *before* the CEI effects. This is acceptable because:
1. `ReentrancyGuardTransient` blocks any re-entrant call from the OO.
2. `settleAndGetPrice` is required to get the price — there is no alternative.
3. The OO is a trusted, audited UMA contract.

---

### 5.4 `priceDisputed(identifier, timestamp, ancillaryData, refund)` — OO Callback

**Access:** `onlyOptimisticOracle` (checks `_knownOracles[msg.sender]`)

The modifier accepts callbacks from any OO ever assigned to this adapter, not only the current global `optimisticOracle`. This is required because a `priceDisputed` callback for a question initialized before an oracle upgrade will arrive from the old OO address, which is a known-but-no-longer-current oracle.

```
// ancillaryData echoed by the OO is fullAncillaryData (raw JSON ++ initializer suffix)
bytes32 questionId = keccak256(ancillaryData)    // == keccak256(fullAncillaryData)
QuestionData storage qd = questions[questionId]

if qd.resolved:                    // Admin already resolved manually
    if qd.reward > 0:
        _bestEffortRefund(questionId, qd)   // try/catch — cannot allow creator blacklist
    return                                  // to revert the disputer's OO transaction

if qd.reset:                       // Second dispute → DVM
    qd.refund = true
    emit QuestionEscalatedToDVM(questionId)  // indexers/bots detect 48–96h DVM window
    return

// First dispute → reset
_resetQuestion(address(this), questionId, false, qd)
```

**Why `address(this)` pays for the reset:** On a first dispute, the reward already on the adapter is reused for the new OO request (the adapter is the `requestor`). On a second dispute (`qd.refund = true`), the reward is returned to the creator upon final resolution.

---

### 5.5 `flag(questionId)` + `unflag(questionId)` + `resolveManually(questionId, winningOutcome)`

**Access:** `EMERGENCY_ROLE` + `nonReentrant`

**`flag`:**
```
check: initialized, not already flagged, not resolved
qd.manualResolveAt = block.timestamp + SAFETY_PERIOD
// NOTE: qd.paused is NOT touched. Flag and pause are orthogonal states.
emit QuestionFlagged
```

**`unflag`:**
```
check: initialized, flagged, not resolved, block.timestamp < manualResolveAt
qd.manualResolveAt = 0
// NOTE: qd.paused is NOT touched. A pauseQuestion() call before or after
// flag/unflag is fully preserved. unflag() can never silently unpause a market.
emit QuestionUnflagged
```

**`resolveManually`:**
```
check: initialized, flagged (manualResolveAt > 0), not resolved
check: block.timestamp >= manualResolveAt
check: winningOutcome < qd.outcomeCount

// CEI
qd.resolved = true

// Critical interaction first — market settlement must not be blocked
IBlieverMarket(qd.market).resolve(winningOutcome)

// Secondary interaction — non-reverting best-effort refund
// try/catch internally; emits RefundFailed if creator cannot receive token.
if qd.refund && qd.reward > 0: _bestEffortRefund(questionId, qd)

emit QuestionManuallyResolved(questionId, winningOutcome)
```

`resolveManually` is reachable both from an admin-initiated `flag()` and from the automatic `TOO_EARLY + reward > 0` flag set by `_decodeAndResolve`. In both cases the guard conditions are identical: `_isFlagged(qd)` and `block.timestamp >= qd.manualResolveAt`.

---

### 5.6 `reset(questionId)`

**Access:** `EMERGENCY_ROLE` + `nonReentrant`  
**Use cases:**
- `priceDisputed` callback failed to fire (OO-side bug). Admin manually issues a fresh OO request.
- `TOO_EARLY + reward > 0` flagged the question. Admin funds a new OO request from their own balance.

```
check: initialized, not resolved
if qd.refund && qd.reward > 0: _refund(qd)

// If flagged (e.g. from TOO_EARLY + reward > 0 path), clear the flag
// before resetting so the fresh request can proceed normally.
if _isFlagged(qd):
    qd.manualResolveAt = 0
    emit QuestionUnflagged

_resetQuestion(msg.sender, questionId, true, qd)
```

`msg.sender` (EMERGENCY_ROLE) pays for the new OO request reward, not `address(this)`. This is the safe recovery path when the adapter's token balance was consumed by a TOO_EARLY settlement.

---

### 5.7 `_resetQuestion(requestor, questionId, resetRefund, qd)`

```
qd.requestTimestamp = uint40(block.timestamp)   // New OO anchor timestamp
qd.reset = true
if resetRefund: qd.refund = false

// Re-snapshot the active oracle for this question's new request.
// Future _hasPrice and settleAndGetPrice calls route to the current
// global optimisticOracle, which may differ from the original if
// updateOptimisticOracle was called between init and this reset.
_questionOracle[questionId] = optimisticOracle

_requestPrice(requestor, newTimestamp, qd.ancillaryData, …)
emit QuestionReset
```

**Critical:** `qd.requestTimestamp` must be updated before `_requestPrice()` is called. The OO identifies the request by `(requester, identifier, timestamp, ancillaryData)` — the new timestamp creates a distinct request key, not an update to the old one. `qd.ancillaryData` (fullAncillaryData) is reused unchanged across resets, keeping every OO request keyed to the same attributed data.

After `_resetQuestion`, the question's `_questionOracle` snapshot points to the current global oracle. If the global oracle changed between this question's init and its first dispute, the reset correctly re-pins to the new oracle, and the old oracle's callback for the now-expired request is handled via `_knownOracles`.

---

### 5.8a `_bestEffortRefund(questionId, qd)` — Non-Reverting Reward Return

Called from all settlement-critical paths: `_decodeAndResolve`, `resolveManually`, and the `qd.resolved` branch of `priceDisputed`.

```
uint256 amount = qd.reward
qd.reward = 0                          // CEI: clear before external call
try IERC20(qd.rewardToken).transfer(qd.creator, amount) returns (bool success) {
    if (!success) {
        emit RefundFailed(questionId, qd.creator, amount, qd.rewardToken)
    }
} catch {
    emit RefundFailed(questionId, qd.creator, amount, qd.rewardToken)
}
```

**Why raw `.transfer()` not `safeTransfer`:**  
`safeTransfer` is injected by the `using SafeERC20 for IERC20` directive as an *internal* function. Solidity's `try/catch` only wraps genuine *external* ABI calls. Wrapping an internal library call in `try/catch` does not compile. The raw `IERC20.transfer()` is a proper external call on the token contract and can be wrapped.

**Why the return bool is explicitly captured:**  
ERC-20 is not uniform in its failure behaviour. Compliant tokens revert on failure, which lands in the `catch` block. Non-standard tokens (older or bespoke implementations) may return `false` without reverting. Without capturing the return value in the `returns (bool success)` clause, a `false` return would pass through the `try` block silently — the `catch` would not fire, no event would be emitted, and the stuck funds would be invisible to monitoring. Capturing and checking `success` ensures `RefundFailed` is emitted in both failure modes.

**CEI note:** `qd.reward` is zeroed before the external `transfer`. This is belt-and-suspenders practice: `ReentrancyGuardTransient` and `qd.resolved = true` already block reentry on all calling paths, but clearing state before external calls is correct regardless.

**Recovery:** If either failure path fires, tokens remain on the adapter. Admins locate the stuck amount via `RefundFailed` logs and recover via a future upgrade or dedicated rescue function.

---

### 5.8b `_refund(qd)` — Hard-Reverting Reward Return

Called **only** from `reset()` — the admin failsafe path where the market is not yet settled and reverting is acceptable. If the creator cannot receive the token, the admin must address the root cause (e.g. token blacklist, pause) before retrying `reset()`.

Do **not** call `_refund` from any settlement path. Use `_bestEffortRefund` instead.

```
IERC20(qd.rewardToken).safeTransfer(qd.creator, qd.reward)
```

---

### 5.9 `updateOptimisticOracle(newOracle)`

**Access:** `DEFAULT_ADMIN_ROLE` + `nonReentrant`

```
if newOracle == address(0): revert ZeroAddress
address oldOracle = address(optimisticOracle)
optimisticOracle         = IOptimisticOracleV2(newOracle)
_knownOracles[newOracle] = true
emit OptimisticOracleUpdated(oldOracle, newOracle)
```

After this call:
- New `initializeQuestion` calls snapshot `_questionOracle[questionId] = newOracle`.
- Existing questions retain their `_questionOracle` snapshot pointing to `oldOracle`. Their `hasPrice` and `settleAndGetPrice` calls continue routing to `oldOracle`.
- `priceDisputed` callbacks from `oldOracle` continue to be accepted because `_knownOracles[oldOracle]` was set to `true` when `oldOracle` was originally assigned.
- **This means there is no window during which an oracle upgrade can brick the resolution path for any live question.** The upgrade is safe at any time regardless of how many unresolved questions exist.

---

## 6. `_hasPrice` and `_ready`

```solidity
function _hasPrice(bytes32 questionId, QuestionData storage qd) internal view returns (bool) {
    return _questionOracle[questionId].hasPrice(
        address(this),
        MULTIPLE_VALUES_IDENTIFIER,
        qd.requestTimestamp,    // Uses the LATEST requestTimestamp (updated on reset)
        qd.ancillaryData        // fullAncillaryData — matches the OO request key
    );
}

function _ready(bytes32 questionId, QuestionData storage qd) internal view returns (bool) {
    if (!_isInitialized(qd)) return false;
    if (qd.paused)           return false;
    if (_isFlagged(qd))      return false;   // Flagged blocks ready() independently of paused
    if (qd.resolved)         return false;
    if (qd.unresolvable)     return false;
    return _hasPrice(questionId, qd);
}
```

Both functions take `questionId` as a parameter to look up `_questionOracle[questionId]`. Routing `hasPrice` through the per-question oracle snapshot instead of the global `optimisticOracle` ensures that `ready()` returns the correct answer even after an oracle upgrade.

`_ready` checks `_isFlagged(qd)` as an independent gate. A question that is flagged but not paused, or paused but not flagged, is correctly gated in both cases.

---

## 7. Role Enforcement — Modifier Summary

| Modifier | Enforced by | Applied on |
|---|---|---|
| `onlyRole(FACTORY_ROLE)` | OZ AccessControl | `initializeQuestion` |
| `onlyRole(EMERGENCY_ROLE)` | OZ AccessControl | `flag`, `unflag`, `resolveManually`, `reset`, `pauseQuestion`, `unpauseQuestion` |
| `onlyRole(DEFAULT_ADMIN_ROLE)` | OZ AccessControl | `pause`, `unpause`, `updateOptimisticOracle` |
| `onlyOptimisticOracle` | `_knownOracles[msg.sender]` | `priceDisputed` callback |
| `whenNotPaused` | OZ Pausable | `initializeQuestion`, `resolve` |
| `nonReentrant` | OZ ReentrancyGuardTransient (EIP-1153) | All state-mutating externals |

---

## 8. UUPS Upgrade Notes

`_authorizeUpgrade(newImplementation)` is gated by `DEFAULT_ADMIN_ROLE`.

**Storage invariants across upgrades:**
- `QuestionData` struct field order must not change.
- `BulletinBoard.updates` mapping slot must not change (inherited first in the linearization after Initializable).
- `optimisticOracle`, `questions`, `_knownOracles`, and `_questionOracle` slots must not change.
- Adding new storage variables is safe only by appending (do NOT insert between existing variables or inherited contracts).

**Safe upgrade pattern:** Use OpenZeppelin's storage gap pattern (`uint256[50] private __gap`) in any new base contracts added in V2 to reserve future slots.

---

## 9. Integration Notes for the MarketFactory

The factory's `deployMarket()` transaction must:

1. Deploy the EIP-1167 clone with `resolver = address(adapter)`.
2. Call `market.initialize(…, _resolver = address(adapter), …)`.
3. Call `adapter.initializeQuestion(questionId, market, ancillaryData, rewardToken, reward, bond, liveness)`.
   - Factory must pre-approve `adapter` as spender for `reward` amount of `rewardToken`.

The `questionId` passed to both `market.initialize` and `adapter.initializeQuestion` **must** equal `keccak256(fullAncillaryData)`, where:

```
fullAncillaryData = abi.encodePacked(ancillaryData, ",initializer:", hexAddress(factoryAddress))
```

`AncillaryDataLib._appendAncillaryData` performs this concatenation on-chain inside `initializeQuestion`. The factory must replicate the same operation off-chain to pre-compute a matching `questionId`.

**Off-chain computation (TypeScript / ethers.js):**
```typescript
const rawJson       = ethers.utils.toUtf8Bytes(jsonString);
const suffix        = ethers.utils.toUtf8Bytes(",initializer:" + factoryAddress.slice(2).toLowerCase());
const fullData      = ethers.utils.concat([rawJson, suffix]);
const questionId    = ethers.utils.keccak256(fullData);
```

The raw `ancillaryData` length must be ≤ `MAX_ANCILLARY_DATA − INITIALIZER_SUFFIX_LENGTH` (8086 bytes).  
`initializeQuestion` will revert with `InvalidAncillaryData` if this is exceeded.

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

### Checking the oracle instance for a question:

```solidity
address oo = adapter.getQuestionOracle(questionId);
// Returns the OO instance this question's price request was submitted to.
// Useful for direct OO inspection (e.g. checking bond amounts, dispute state).
```

---

## 10. Key Error Conditions and Debug Guide

| Error | Cause | Fix |
|---|---|---|
| `QuestionIdMismatch` | `keccak256(fullAncillaryData) != questionId` in `initializeQuestion` | Recompute questionId off-chain from fullAncillaryData (raw JSON ++ `",initializer:"` ++ lower-case hex factory address) |
| `QuestionIdMismatch` | `market.questionId() != questionId` | Market was initialized with wrong questionId; redeploy |
| `AlreadyInitialized` | `initializeQuestion` called twice for same questionId | Check `isInitialized(questionId)` before calling |
| `InvalidAncillaryData` | `ancillaryData.length == 0` or raw length > `MAX_ANCILLARY_DATA − INITIALIZER_SUFFIX_LENGTH` (8086 bytes) | Shorten the raw JSON; the 53-byte initializer suffix is appended on-chain |
| `InvalidOutcomeCount` | `market.outcomeCount() > 7` | V1 cap; use outcomeCount ∈ [2, 7] |
| `PriceNotAvailable` | `resolve()` called before liveness window closed | Check `ready(questionId)` first; `resolve()` will propagate the OO's native revert if the price is not settled |
| `Flagged` | `resolve()` called while question is flagged for manual resolution | Wait for EMERGENCY_ROLE to call `resolveManually()`, or unflag if within safety period |
| `NotOptimisticOracle` | `priceDisputed` called by an address not in `_knownOracles` | OO address was never registered; check `_knownOracles` via a fork test |
| `InvalidOracleEncoding` | Oracle returned a price with multiple winners, a value > 1, or non-zero trailing bits | Review proposer bot encoding; ensure it uses `encodeValues([0,..,1,..,0])` |
| `Unresolvable` | `resolve()` called after event was canceled (oracle returned int256.max) | Check `getQuestion(id).unresolvable`; trigger `factory.expireUnresolved()` |
| `SafetyPeriodNotPassed` | `resolveManually` called too early | Wait until `block.timestamp >= getQuestion(id).manualResolveAt` |
| `SafetyPeriodPassed` | `unflag()` called after the safety window closed | Cannot unflag; must proceed with `resolveManually` or `reset` |

---

## 11. Gas Notes

- `initializeQuestion` is the most gas-intensive call: appends initializer suffix (pure computation, ~2k gas), writes 6+ storage slots + `_questionOracle` snapshot, + 3–5 OO external calls. Expected ~250k–310k gas on Base.
- `resolve()` (fast path, no dispute): ~77k–127k gas. `_questionOracle` lookup (warm SLOAD) + `settleAndGetPrice` + `decodeWinningOutcome` (pure math) + `market.resolve()` (1 SSTORE resolved + pool.settleMarket). No `OO.hasPrice()` view call — removed as redundant.
- `priceDisputed` callback (first dispute): ~65k gas. Two SSTOREs + `_questionOracle` re-snapshot + one new OO request.
- `priceDisputed` callback (second dispute → DVM): ~25k gas. One SSTORE (`qd.refund = true`) + one event (`QuestionEscalatedToDVM`). No new OO request.
- `_bestEffortRefund` (success path): ~30k gas. One SSTORE (`qd.reward = 0`) + one external `transfer`. On non-reverting false return: same SSTORE + one `RefundFailed` event (~3k gas for the log). On hard revert (catch): same SSTORE + one `RefundFailed` event.
- `_resetQuestion`: re-snapshots `_questionOracle[questionId] = optimisticOracle` — one warm SSTORE, negligible cost.
- `postUpdate` (BulletinBoard): ~25k gas (one SSTORE for the new array element).
- `ReentrancyGuardTransient` (EIP-1153): ~0 persistent gas overhead vs ~20k for the classic guard. Requires EIP-1153 support (Base supports this from genesis).
- `onlyOptimisticOracle` modifier: uses `_knownOracles[msg.sender]` (one SLOAD) instead of a direct address comparison. Cost is identical on the hot path since the SLOAD is warm for the current OO after `updateOptimisticOracle`.
