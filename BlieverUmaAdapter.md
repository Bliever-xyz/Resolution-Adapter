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

### 2.2 Modular Oracle Coupling

The adapter treats the UMA oracle as a pluggable dependency. The `optimisticOracle` address can be updated by governance without touching any market contract. When UMA ships a new oracle version:

1. `DEFAULT_ADMIN_ROLE` calls `updateOptimisticOracle(newOO)` on the adapter proxy.
2. All new questions use the new oracle. Already-settled questions are unaffected.
3. Zero market migrations, zero re-deployments.

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
│   │  Adapter issues fresh OO.requestPrice (new timestamp)        │  │
│   │  Bot must propose again within the new liveness window       │  │
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
│     OO.settleAndGetPrice → int256 encodedPrice                      │
│     int256.min?   → reset question (too early)                      │
│     int256.max?   → mark unresolvable; factory expires market       │
│     valid price → decodeWinningOutcome → market.resolve(winner)     │
│     market.resolve → pool.settleMarket(totalPayoutUsdc)             │
│     Winners call market.claim() → pool.claimWinnings()              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Emergency Path — Admin Circuit Breaker

The DVM is secured by game theory (corrupting it requires > 50% of active voting UMA). However, the "Ukraine Mineral Agreement" incident (March 2025) demonstrated that a minority of highly concentrated active voters can produce a factually incorrect DVM result in a thinly-attended vote.

The adapter's emergency path allows the **EMERGENCY_ROLE** (protocol DAO multisig) to intercept before a bad result propagates:

```
EMERGENCY_ROLE calls flag(questionId)
  → qd.paused = true  (blocks permissionless resolve())
  → qd.manualResolveAt = now + 1 hour  (public 1-hour notice window)

After 1 hour:
EMERGENCY_ROLE calls resolveManually(questionId, correctWinner)
  → Bypasses OO entirely
  → Calls market.resolve(correctWinner) directly
```

The 1-hour delay is a deliberate transparency mechanism: it gives the community time to contest the admin's decision before it is irreversible. `unflag()` can cancel the intervention before the period ends.

---

## 6. BulletinBoard — On-Chain Dispute Quality

Prediction markets frequently have ambiguous resolution criteria. Example: "Will the US Federal Reserve cut rates by Q2 2026?" — what counts as a "cut"? 25 bps? 50 bps? An emergency cut?

`BulletinBoard` (sourced unchanged from Polymarket's battle-tested repository) allows the question creator — or any interested party — to post on-chain clarifications:

- Keyed by `keccak256(abi.encode(questionId, poster))`.
- UMA DVM voters are bound to consider updates from the **question creator's address**.
- Zero impact on the trading or resolution hot paths — `postUpdate()` is never called in normal settlement.

`AncillaryDataLib` (also Polymarket-sourced) complements this by embedding the factory address directly inside the ancillary data submitted to UMA. DVM voters see `,initializer:0xFactory…` in every request, unambiguously attributing the market to the Bliever factory rather than an anonymous address.

Both patterns have been proven in production on Polymarket's millions of markets. Adding them costs almost nothing; removing either later would require an upgrade.

---

## 7. Access Control Matrix

| Role | Who Holds It | Permissions |
|---|---|---|
| `DEFAULT_ADMIN_ROLE` | Protocol governance multisig | `pause()` globally, `unpause()`, `updateOptimisticOracle()`, authorize UUPS upgrades, grant/revoke all roles |
| `FACTORY_ROLE` | `BlieverMarketFactory` contract | `initializeQuestion()` — the only address that can register a new question |
| `EMERGENCY_ROLE` | Emergency multisig / DAO | `flag()`, `unflag()`, `resolveManually()`, `reset()`, `pauseQuestion()`, `unpauseQuestion()` |

`resolve()` and `BulletinBoard.postUpdate()` are **permissionless** — any EOA can call them.  
The OO callback `priceDisputed()` is gated by `onlyOptimisticOracle` (current OO address).

---

## 8. Upgrade Model

The adapter is a **UUPS proxy**. The upgrade mechanism is:
- `_authorizeUpgrade()` gated to `DEFAULT_ADMIN_ROLE` (governance multisig + timelock in production).
- The stored `QuestionData` mapping persists across upgrades (storage layout is fixed by the `QuestionData` struct — do NOT reorder fields).
- Individual `BlieverMarket` clones are immutable EIP-1167 — they will continue pointing to the same adapter proxy address, which seamlessly receives the upgraded logic.

---

## 9. Security Considerations

| Risk | Mitigation |
|---|---|
| Re-entrancy via `market.resolve()` | `ReentrancyGuardTransient` (EIP-1153) on all state-mutating external paths |
| Malicious OO callback | `onlyOptimisticOracle` modifier on `priceDisputed()`; checks current `optimisticOracle` address |
| Duplicate resolution | `qd.resolved` flag checked first in `resolve()` and `resolveManually()` |
| Admin governance attack (flag + resolveManually with wrong outcome) | 1-hour SAFETY_PERIOD public delay; community can contest; upgrade path is also gated by timelock |
| questionId collision / mismatch | `keccak256(fullAncillaryData) == questionId` AND `market.questionId() == questionId` both verified in `initializeQuestion()`; `fullAncillaryData` is computed on-chain by `AncillaryDataLib._appendAncillaryData` |
| Oracle encoding exploit (crafted int256) | `MultiValueDecoder.decodeWinningOutcome()` enforces: top bits = 0, trailing slots = 0, exactly one slot = 1, all others = 0; reverts otherwise |
| OO version upgrade breaking historical requests | Historical requests retain their `requestTimestamp` key; only NEW requests use the updated OO address |
| Reward token stuck in adapter | `_refund()` triggered on `qd.refund == true` in both `_decodeAndResolve()` and `resolveManually()` |
