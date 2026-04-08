# BlieverUmaAdapter — Architecture & System Design

> **Document type:** Theoretical architecture, system design, and rationale.  
> **Audience:** Anyone who wants to understand *what* the contract does and *why* it is designed this way — protocol designers, auditors, investors, integrators.  
> **Companion:** `BlieverUmaAdapter_DEV.md` covers the technical implementation details.

---

## 1. Context — Where the Adapter Sits in the Stack

The Bliever Protocol is a multi-contract prediction market system. Each layer has a single, clearly bounded responsibility:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Bliever Protocol Stack                          │
├────────────────────────────────────────────────────────────────────────┤
│  BlieverMarketFactory (off-chain CREATE2 + EIP-1167)                   │
│    Deploys clones, registers questions with the adapter.               │
├────────────────────────────────────────────────────────────────────────┤
│  BlieverUmaAdapter  ◄── YOU ARE HERE                                   │
│    Oracle Request ↔ MULTIPLE_VALUES decode → market.resolve()          │
│    BulletinBoard on-chain clarification registry                        │
├────────────────────────────────────────────────────────────────────────┤
│  BlieverMarket (EIP-1167 clone × N markets)                            │
│    LS-LMSR AMM, internal share ledger, trading, resolve(), claim()      │
├────────────────────────────────────────────────────────────────────────┤
│  BlieverV1Pool (single ERC-4626 vault)                                 │
│    All USDC, LP NAV, liability tracking, settleMarket(), claimWinnings()│
├────────────────────────────────────────────────────────────────────────┤
│  LSMath (pure library)                                                 │
│    LS-LMSR cost, price, worst-case-loss — no state                     │
└────────────────────────────────────────────────────────────────────────┘
```

The adapter is the **sole bridge between the UMA oracle network and the BlieverMarket contracts**. It is stored as `resolver` in every BlieverMarket clone; only the adapter can call `market.resolve(winningOutcome)`.

---

## 2. Design Principles

### 2.1 No Duplication of Existing State

The adapter does **not** hold USDC, track shares, maintain liability counters, or own any AMM state. All of that lives in `BlieverV1Pool` and `BlieverMarket`. The adapter only stores oracle-coordination state — `QuestionData` — per registered question.

### 2.2 Modular Oracle Coupling — Per-Question Oracle Tracking

The adapter treats the UMA oracle as a pluggable dependency. The `optimisticOracle` global address can be updated by governance without touching any market contract. When UMA ships a new oracle version:

1. `DEFAULT_ADMIN_ROLE` calls `updateOptimisticOracle(newOO)` on the adapter proxy.
2. All new questions use the new oracle. Their `_questionOracle` snapshot is recorded at `initializeQuestion` time and routes their `hasPrice` and `settleAndGetPrice` calls to `newOO`.
3. Already-initialized questions are unaffected. Each question stores a `_questionOracle` snapshot pointing to the OO instance it was originally submitted to. Resolution calls for those questions continue routing to the original OO, not `newOO`.
4. `priceDisputed` callbacks from the old OO are accepted via the `_knownOracles` registry, which records every OO address ever assigned to the adapter. The `onlyOptimisticOracle` modifier checks this registry rather than comparing against the single current OO address.
5. **Oracle upgrades are safe at any time, even while hundreds of live questions are in-flight.** No markets are bricked.

This is the core justification for making the adapter an upgradeable UUPS proxy while keeping all markets as immutable EIP-1167 clones.

### 2.3 V1 Outcome Cap — MAX_OUTCOMES = 7

UMIP-183 supports up to 7 labels. V1 enforces this as a hard limit for simplicity, gas efficiency, and to guarantee that all markets use a single, well-understood oracle identifier (`MULTIPLE_VALUES`) with no fallback path required. 100-outcome markets can be added in V2.

---

## 3. Oracle Choice — UMA Optimistic Oracle (MULTIPLE_VALUES)

### 3.1 Why not Chainlink or Pyth?

Price-feed oracles are optimized for high-frequency numeric data (ETH/USD, BTC price). They cannot resolve intersubjective truths like "Who will win the 2026 U.S. presidential election?" The UMA Optimistic Oracle (OO) is purpose-built for exactly this: it allows any data point to be proposed on-chain with a financial bond and challenged by anyone who believes it is incorrect.

### 3.2 Why ManagedOptimisticOracleV2?

The standard OOv2 is permissionless for proposals — anyone can propose, causing spam from low-bond actors that triggers dispute loops and delays settlement by 48–96 hours per dispute. The `ManagedOptimisticOracleV2` adds a whitelisted proposer layer:

- **`REQUEST_MANAGER_ROLE`** on the managed OO restricts the initial proposer to a vetted set of protocol bots.
- **Dispute right** remains **permissionless** — anyone can challenge a bad proposal.
- **Result:** ~2-hour fast path for legitimate outcomes; full DVM arbitration for genuine disputes.

### 3.3 Why MULTIPLE_VALUES (UMIP-183)?

For a market with N outcomes, the legacy approach required N separate `YES_OR_NO` oracle requests — N liveness windows, N bond commitments, N opportunities for a dispute to stall the entire settlement. UMIP-183 allows a single oracle request to return the full payout vector encoded in one `int256`:

```
int256
| bits 224–255 | bits 192–223 | … | bits 0–31   |
| always zero  | outcome 6    | … | outcome 0   |
```

For a binary market: `encodedPrice = 1 << (32 * winnerIndex)`. For outcome 0 winning: `1`. For outcome 1 winning: `4294967296` (= `1 << 32`).

The adapter calls `MultiValueDecoder.decodeWinningOutcome()` to extract the index of the outcome with value = 1.

---

## 4. Three-Phase Oracle Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 1 — INITIALIZATION                                            │
│   Factory calls initializeQuestion(questionId, market, ancData, …) │
│   Adapter → AncillaryDataLib.append(factory, ancData) → fullData   │
│   Adapter → OO.requestPrice(MULTIPLE_VALUES, fullData, …)          │
│   Adapter → OO.setEventBased(…)                                     │
│   Adapter → OO.setCallbacks(…, disputeCallback=true, …)            │
│   Adapter → OO.setBond / setCustomLiveness (if overrides provided) │
│   Adapter snapshots _questionOracle[questionId] = optimisticOracle  │
└─────────────────────────────────────────────────────────────────────┘
                         │
                         ▼  (event happens in the real world)
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 2 — PROPOSAL WINDOW (2-hour liveness)                         │
│   Whitelisted bot calls OO.proposePriceFor(encodedWinner, …)        │
│   ┌──────────── No dispute ──────────────────────────────────────┐  │
│   │  2 hours pass → OO settles the proposal                     │  │
│   │  Any EOA calls adapter.resolve(questionId) → fast path      │  │
│   └──────────────────────────────────────────────────────────────┘  │
│   ┌──────────── First dispute ───────────────────────────────────┐  │
│   │  OO calls adapter.priceDisputed() callback                  │  │
│   │  Timestamp guard: stale callback? → silent return (no-op)   │  │
│   │  try _tryResetQuestion → fresh OO request, new timestamp     │  │
│   │    success: bot must propose again within new liveness       │  │
│   │    catch:   flag for EMERGENCY_ROLE recovery via reset()     │  │
│   └──────────────────────────────────────────────────────────────┘  │
│   ┌──────────── Second dispute → DVM ───────────────────────────┐  │
│   │  priceDisputed sets qd.refund = true, no new OO request     │  │
│   │  UMA token-holders vote (48–96 hours)                       │  │
│   │  Any EOA calls adapter.resolve(questionId) after DVM done   │  │
│   └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 3 — SETTLEMENT                                                │
│   adapter.resolve(questionId):                                      │
│     _questionOracle[questionId].settleAndGetPrice → int256          │
│     int256.min + reward == 0 → reset question (too early)           │
│     int256.min + reward  > 0 → flag for admin; reward was consumed  │
│     int256.max?   → mark unresolvable; factory expires market       │
│     valid price → decodeWinningOutcome → market.resolve(winner)     │
│     market.resolve → pool.settleMarket(totalPayoutUsdc)             │
│     _bestEffortRefund if applicable — non-reverting, after settle   │
│     Winners call market.claim() → pool.claimWinnings()              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Emergency Path — Admin Circuit Breaker

The DVM is secured by game theory (corrupting it requires > 50% of active voting UMA). However, the "Ukraine Mineral Agreement" incident (March 2025) demonstrated that a minority of highly concentrated active voters can produce a factually incorrect DVM result in a thinly-attended vote.

The adapter's emergency path allows the **EMERGENCY_ROLE** (protocol DAO multisig) to intercept before a bad result propagates:

```
EMERGENCY_ROLE calls flag(questionId)
  → qd.manualResolveAt = now + 1 hour  (public 1-hour notice window)
  → resolve() reverts with Flagged (independent of qd.paused)

After 1 hour:
EMERGENCY_ROLE calls resolveManually(questionId, correctWinner)
  → Bypasses OO entirely
  → Calls market.resolve(correctWinner) directly
  → _bestEffortRefund if applicable — non-reverting, after market.resolve()
  → emits QuestionManuallyResolved(questionId, winningOutcome, msg.sender)
     ↑ msg.sender is indexed as `resolver` — permanent on-chain record of
       which admin wallet executed the emergency intervention
```

The 1-hour delay is a deliberate transparency mechanism: it gives the community time to contest the admin's decision before it is irreversible. `unflag()` can cancel the intervention before the period ends.

**Pause and flag are independent states.** `flag()` and `unflag()` operate only on `qd.manualResolveAt` and never touch `qd.paused`. A question explicitly paused via `pauseQuestion()` before or after a flag/unflag cycle retains its paused state exactly as set. There is no state-overwrite interaction between the two mechanisms.

---

## 5a. Dispute Callback Resilience

The `priceDisputed` callback is the only code path the adapter cannot initiate — it is driven entirely by the UMA OO calling back into the adapter when a disputer posts their bond. Two defensive mechanisms protect the integrity of this path.

**Timestamp guard.** The OO echoes back the `timestamp` parameter that was originally set on the request. On a first dispute, the adapter already called `_resetQuestion`, which updates `qd.requestTimestamp` to a new value. If a delayed or replayed callback for the *old* request arrives afterward, the guard `if (timestamp != qd.requestTimestamp) return` exits silently. Without this guard, a late callback could trigger an erroneous second reset on a question whose new request is already in-flight.

**Non-reverting reset via try/catch.** When `_resetQuestion` is called during a first dispute it routes through `_requestPrice`, which transfers tokens and calls the OO. Either of these external calls can revert — for example if the reward token is paused, or if the OO itself is temporarily paused by UMA governance. If `_resetQuestion` propagated a revert, the disputer's entire OO transaction would fail. The disputer would lose the ability to post their bond and the bad proposal would finalize unchallenged.

The adapter instead wraps the reset in `try this._tryResetQuestion(...) { } catch { }`. If the reset reverts, the catch block sets `qd.manualResolveAt` and emits `QuestionFlagged`, exactly as the TOO_EARLY + reward > 0 path does. EMERGENCY_ROLE then calls `reset(questionId)` to issue a fresh OO request from its own balance. The disputer's transaction always lands; the market never gets locked by a reset failure.

`_tryResetQuestion` is declared `external` solely to satisfy Solidity's constraint that `try/catch` only wraps genuine external ABI calls. It is access-guarded: `if (msg.sender != address(this)) revert BlieverUmaAdapter__OnlySelf()`. Only the adapter itself (via `this.X`) can invoke it.

---

## 6. TOO_EARLY Recovery Path

When UMA's event-based oracle receives a proposal before the event anchor timestamp, it settles with `type(int256).min` (TOO_EARLY_PRICE). The adapter's response branches on whether a reward was attached to the question:

**`reward == 0` (common V1 case):** The adapter auto-resets — it issues a fresh OO price request with a new timestamp. No token transfer is involved so there is no balance risk.

**`reward > 0`:** UMA consumes the reward token when settling a TOO_EARLY result. It does not refund it to the adapter. The adapter cannot safely self-fund a new request from a zero balance. Instead, the adapter sets `qd.manualResolveAt = now + SAFETY_PERIOD` and emits `QuestionFlagged`. EMERGENCY_ROLE then calls `reset(questionId)`, paying for the new OO request from their own balance, which also clears the flag atomically.

---

## 7. BulletinBoard — On-Chain Dispute Quality

Prediction markets frequently have ambiguous resolution criteria. Example: "Will the US Federal Reserve cut rates by Q2 2026?" — what counts as a "cut"? 25 bps? 50 bps? An emergency cut?

`BulletinBoard` (sourced unchanged from Polymarket's battle-tested repository) allows the question creator — or any interested party — to post on-chain clarifications:

- Keyed by `keccak256(abi.encode(questionId, poster))`.
- UMA DVM voters are bound to consider updates from the **question creator's address**.
- Zero impact on the trading or resolution hot paths — `postUpdate()` is never called in normal settlement.

`AncillaryDataLib` (also Polymarket-sourced) complements this by embedding the factory address directly inside the ancillary data submitted to UMA. DVM voters see `,initializer:0xFactory…` in every request, unambiguously attributing the market to the Bliever factory rather than an anonymous address.

Both patterns have been proven in production on Polymarket's millions of markets. Adding them costs almost nothing; removing either later would require an upgrade.

---

## 8. Access Control Matrix

| Role | Who Holds It | Permissions |
|---|---|---|
| `DEFAULT_ADMIN_ROLE` | Protocol governance multisig | `pause()` globally, `unpause()`, `updateOptimisticOracle()`, authorize UUPS upgrades, grant/revoke all roles |
| `FACTORY_ROLE` | `BlieverMarketFactory` contract | `initializeQuestion()` — the only address that can register a new question |
| `EMERGENCY_ROLE` | Emergency multisig / DAO | `flag()`, `unflag()`, `resolveManually()`, `reset()`, `pauseQuestion()`, `unpauseQuestion()` |

`resolve()` and `BulletinBoard.postUpdate()` are **permissionless** — any EOA can call them.  
The OO callback `priceDisputed()` is gated by `onlyOptimisticOracle` (any address in `_knownOracles`).

---

## 9. Upgrade Model

The adapter is a **UUPS proxy**. The upgrade mechanism is:
- `_authorizeUpgrade()` gated to `DEFAULT_ADMIN_ROLE` (governance multisig + timelock in production).
- The stored `QuestionData` mapping persists across upgrades (storage layout is fixed by the `QuestionData` struct — do NOT reorder fields).
- `_knownOracles` and `_questionOracle` mappings also persist and must not be relocated.
- Individual `BlieverMarket` clones are immutable EIP-1167 — they will continue pointing to the same adapter proxy address, which seamlessly receives the upgraded logic.

---

## 10. Security Considerations

| Risk | Mitigation |
|---|---|
| Re-entrancy via `market.resolve()` | `ReentrancyGuardTransient` (EIP-1153) on all state-mutating external paths |
| Malicious OO callback | `onlyOptimisticOracle` modifier gates `priceDisputed()`; checks `_knownOracles[msg.sender]` — only addresses ever legitimately assigned as OO are accepted |
| Duplicate resolution | `qd.resolved` flag checked first in `resolve()` and `resolveManually()` |
| Admin governance attack (flag + resolveManually with wrong outcome) | 1-hour SAFETY_PERIOD public delay; community can contest; `QuestionManuallyResolved` indexes `resolver` (= `msg.sender`) providing an immutable on-chain record of which admin wallet acted; upgrade path is also gated by timelock |
| questionId collision / mismatch | `keccak256(fullAncillaryData) == questionId` AND `market.questionId() == questionId` both verified in `initializeQuestion()`; `fullAncillaryData` is computed on-chain by `AncillaryDataLib._appendAncillaryData` |
| Oracle encoding exploit (crafted int256) | `MultiValueDecoder.decodeWinningOutcome()` enforces: top bits = 0, trailing slots = 0, exactly one slot = 1, all others = 0; reverts otherwise |
| OO version upgrade breaking historical requests | Each question's `_questionOracle` snapshot pins the OO it was submitted to. Upgrades never affect in-flight questions. `_knownOracles` ensures old-OO callbacks are still accepted. |
| Reward token stuck in adapter | `_bestEffortRefund` captures the bool return from `transfer()` and emits `RefundFailed(questionId, creator, amount, token)` on both hard reverts (e.g. USDC blacklist) and non-reverting false returns (non-standard ERC-20s). Stranded tokens remain on the adapter; admins recover via event log and a future upgrade or rescue function. |
| Reward token blacklist blocking market settlement | `market.resolve()` executes before `_bestEffortRefund()` in all settlement paths. A failed refund emits `RefundFailed` and never prevents winners from claiming. |
| DVM escalation invisible to off-chain systems | `priceDisputed` emits `QuestionEscalatedToDVM(questionId)` on the second dispute. Indexers and monitoring bots listen for this event to detect the 48–96-hour DVM arbitration window without polling oracle state. |
| TOO_EARLY with reward > 0 draining adapter | Adapter does not attempt self-funded reset when `reward > 0`. Sets `manualResolveAt` and flags for admin recovery via `reset()`, which pulls fresh funds from EMERGENCY_ROLE. |
| `unflag()` silently unpausing a `pauseQuestion()`-paused market | `flag()` and `unflag()` never touch `qd.paused`. The two states are fully orthogonal. `resolve()` checks both independently. |
| Stale or replayed OO callback acting on superseded request state | `priceDisputed` guards `if (timestamp != qd.requestTimestamp) return` before any state change. A callback for an old request timestamp silently exits; the current request state is never corrupted. |
| Reset failure in `priceDisputed` blocking dispute economics | `_resetQuestion` is wrapped in `try this._tryResetQuestion(...) catch`. If the reset reverts (e.g. token paused, OO paused), the catch flags the question for admin recovery instead of propagating the revert to the disputer's transaction. The disputer's OO call always lands. |
| `_tryResetQuestion` called externally by an unauthorized address | `_tryResetQuestion` is declared `external` only to satisfy Solidity's try/catch requirement. It is access-guarded: `if (msg.sender != address(this)) revert BlieverUmaAdapter__OnlySelf()`. No external actor can invoke it. |
