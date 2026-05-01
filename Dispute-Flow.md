# Dual Dispute Flow
 
**System:** `BlieverUmaAdapter` ↔ `UMA Optimistic Oracle V2`  
**Document Scope:** How the adapter handles two sequential disputes on the same question, transitioning from optimistic resolution to full UMA DVM arbitration.

---

## Executive Summary

The UMA Optimistic Oracle (OO) allows anyone to dispute a proposed price during the liveness window. Bliever's adapter is designed to handle exactly **two dispute rounds** per question before escalating to the UMA DVM (Data Verification Mechanism), which is a full token-holder vote lasting 48–96 hours.

The two-round design serves a critical purpose: the first dispute gives the system a chance to self-correct (perhaps the bot proposed the wrong outcome by mistake). The second dispute signals genuine disagreement that only decentralized arbitration can resolve.

> **Core Flow:**  
> `Proposal` → `[Dispute 1]` → `Reset + New Proposal` → `[Dispute 2]` → `DVM Arbitration` → `Resolution`

---

## Visual Architecture Overview

The following diagram illustrates the interaction between the Bliever Protocol components and the UMA Ecosystem during the dispute and resolution lifecycle.

![Architecture Component Diagram](https://private-us-east-1.manuscdn.com/sessionFile/RFp897btO1Cqct7PJ3MyE0/sandbox/32bqWcFpETGZpvTOmVvkky-images_1775923013994_na1fn_L2hvbWUvdWJ1bnR1L2FyY2hpdGVjdHVyZQ.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvUkZwODk3YnRPMUNxY3Q3UEozTXlFMC9zYW5kYm94LzMyYnFXY0ZwRVRHWnB2VE9tVnZra3ktaW1hZ2VzXzE3NzU5MjMwMTM5OTRfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyRnlZMmhwZEdWamRIVnlaUS5wbmciLCJDb25kaXRpb24iOnsiRGF0ZUxlc3NUaGFuIjp7IkFXUzpFcG9jaFRpbWUiOjE3OTg3NjE2MDB9fX1dfQ__&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=GPCRObN-RUtqiYEmueBJaCORkFQLFXJGPX9OLPGSDXbw5X9oLhd2Bjpx7AVilAQeZtYfwE0ZrYJf3LFhYCVqp1TvxVpIp0VVf5bLNs8RKLQf7eW9-LAwrcQp4HB53fj9DG~y2iVC2zWA0DSS6Te-3hWlAbseGNmdibq9--HDGgE4eUMZKX9nNzDYOJurP9Pz4T0vRFaiZXERGPlY3lpldNxei4h6-~so9tjcZ7tM-ZUQ63yfE1duj8WtmVJKBBNtUVLbAza-Ihavxb3slbeaNa8OspjsY1HxfTUGxBvmMokmkoC3abe9TSoEkGrjxqeES1moVcytYQ7EqOZKScGHdw__)

---

## Phase 0 — Initial State (Post-initialization)

When `initializeQuestion()` is called by the factory, the adapter sets up the question data (`qd`) and snapshots the active oracle.

| Field | Initial Value | Description |
| :--- | :--- | :--- |
| `qd.requestTimestamp` | `block.timestamp` (T0) | The anchor for the first OO request. |
| `qd.reset` | `false` | Indicates no disputes have occurred yet. |
| `qd.refund` | `false` | No refund is pending. |
| `qd.resolved` | `false` | Question is active. |
| `_questionOracle` | `optimisticOracle` | Snapshot of the active OO instance. |

A price request is live on the OO with:
- **Identifier:** `bytes32("MULTIPLE_VALUES")`
- **Timestamp:** `T0`
- **AncillaryData:** `rawJSON + ",initializer:" + hex(factory)`
- **Event-based mode:** ON
- **Callback:** `priceDisputed` only

---

## Phase 1 — First Dispute

### Trigger
A disputer calls `OO.disputePrice(...)` during the liveness window, posting a bond equal to the proposer's bond.

### Callback: `priceDisputed(identifier, timestamp, ancillaryData, refund)`
The OO fires this callback on the adapter. The adapter's `onlyOptimisticOracle` modifier validates that `msg.sender` is in `_knownOracles`.

#### Critical Guard: Timestamp Check
```solidity
if (timestamp != qd.requestTimestamp) return;
```
This silently discards stale callbacks from superseded requests. It is the most critical guard in the entire callback — without it, a replay of an old dispute callback could accidentally trigger a second-dispute escalation.

#### First Dispute Branch (`qd.reset == false`)
The adapter attempts to reset the question using an external self-call wrapper:
```solidity
try this._tryResetQuestion(address(this), questionId, false) {
    // success — new OO request issued
} catch {
    // failure — flag for admin recovery
    qd.manualResolveAt = uint40(block.timestamp + SAFETY_PERIOD);
    emit QuestionFlagged(questionId);
}
```

### Result of Successful Reset
`_resetQuestion` executes, updating the state for the second attempt:
- `qd.requestTimestamp = block.timestamp` (T1, new anchor)
- `qd.reset = true` (Marks attempt #2)
- `_questionOracle[questionId] = optimisticOracle` (Re-snapshot for upgrade safety)

---

## Phase 2 — Second Dispute

### Trigger
The proposer re-proposes on the new request (T1). A second disputer (or the same one) disputes again.

### Callback: `priceDisputed(identifier, T1, ancillaryData, refund)`
The timestamp guard passes (`timestamp == T1`). Since `qd.reset == true`, the adapter enters the escalation branch:

```solidity
qd.refund = true;
emit QuestionEscalatedToDVM(questionId);
return;
```

No new OO request is issued. The existing managed OO request (now under DVM jurisdiction) stays active. The DVM takes ownership of the arbitration.

> **Monitoring Note:** `QuestionEscalatedToDVM` is the only on-chain signal distinguishing "first dispute reset" from "second dispute DVM escalation." Off-chain bots must listen for this event to enter the 48–96-hour waiting window.

---

## Phase 3 & 4 — DVM Arbitration & Resolution

The UMA DVM proceeds independently. UMA token-holders vote on the correct value. Once the vote concludes, the OO marks the request as settled.

### Resolution Sequence
Any EOA calls `adapter.resolve(questionId)`. The adapter enters `_decodeAndResolve(questionId, qd)`:

1. **Settle:** `_questionOracle[questionId].settleAndGetPrice(MULTIPLE_VALUES, T1, ancillaryData)`
2. **Decode:** `MultiValueDecoder.decodeWinningOutcome(encodedPrice, qd.outcomeCount)` → `uint8 winner`
3. **CEI Effect:** `qd.resolved = true`
4. **Market Interaction:** `IBlieverMarket(qd.market).resolve(winner)`
5. **Refund Path:** If `qd.refund == true`, `_bestEffortRefund(questionId, qd)` is called to return the reward to the creator.

---

## Visual Flow & State Transitions

The following sequence diagram summarizes the entire lifecycle from proposal to DVM resolution.

![Sequence Diagram](https://private-us-east-1.manuscdn.com/sessionFile/RFp897btO1Cqct7PJ3MyE0/sandbox/32bqWcFpETGZpvTOmVvkky-images_1775923013994_na1fn_L2hvbWUvdWJ1bnR1L3NlcXVlbmNl.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvUkZwODk3YnRPMUNxY3Q3UEozTXlFMC9zYW5kYm94LzMyYnFXY0ZwRVRHWnB2VE9tVnZra3ktaW1hZ2VzXzE3NzU5MjMwMTM5OTRfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwzTmxjWFZsYm1ObC5wbmciLCJDb25kaXRpb24iOnsiRGF0ZUxlc3NUaGFuIjp7IkFXUzpFcG9jaFRpbWUiOjE3OTg3NjE2MDB9fX1dfQ__&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=Jsml60yk1abHChlk~oDQfQWlv6TwY2ct6OrTofiyuhqvdgQViXvah5zWDaZjCvh8-CDusoYqI5AB41CKcHjwiVgfwx2o0CjizQHOdMtbnenC0ckedNnpHdL05VdVyMbUbNccyE1egnl7zmkvC6PhIRCPBaVRsyGgGgSVfxlyhIKKEOPYJ4XflRmVnoPaonJ3QdM1XxgeYv-2H7wCZQnx3hfrJbWsZLWnS9DsGLXlCtTcEhi3xt-YjFOUUtwhV6wmTBK40ZpFwNRzk5zq5afkwuacf4x404UbPM1aNckqZfWK96y4wb3eaewSQ~wVXdDm3fm9XTd2-o96tmn5dJuCyQ__)

### State Transition Table

| Event | `reset` | `refund` | `resolved` | `manualResolveAt` |
| :--- | :---: | :---: | :---: | :---: |
| Question initialized | `false` | `false` | `false` | 0 |
| First dispute (reset succeeds) | **`true`** | `false` | `false` | 0 |
| First dispute (reset fails) | `false` | `false` | `false` | **`now+3600`** |
| Second dispute | `true` | **`true`** | `false` | 0 |
| DVM resolves — `resolve()` called | `true` | **`false`** | **`true`** | 0 |

---

## Key Contracts and Functions

| Contract | Function | Role |
| :--- | :--- | :--- |
| `BlieverUmaAdapter` | `priceDisputed()` | OO callback; routes dispute to reset or DVM escalation |
| `BlieverUmaAdapter` | `_tryResetQuestion()` | External self-call wrapper enabling try/catch on reset |
| `BlieverUmaAdapter` | `_resetQuestion()` | Issues fresh OO request; re-snapshots oracle |
| `BlieverUmaAdapter` | `_decodeAndResolve()` | Settles price, decodes winner, calls market |
| `BlieverUmaAdapter` | `_bestEffortRefund()` | Returns reward post-DVM; never reverts |
| `IOptimisticOracleV2` | `requestPrice()` | Submits new OO request on reset |
| `MultiValueDecoder` | `decodeWinningOutcome()` | UMIP-183 decoding with 4-rule validation |
| `IBlieverMarket` | `resolve()` | Final settlement entry point |

---

## Edge Cases and Guards

- **External Self-Call:** `_tryResetQuestion` must be external with an access guard because Solidity `try/catch` requires a genuine external ABI call. This ensures that if `_requestPrice` reverts, the OO's dispute transaction still lands, and the question is flagged for admin recovery.
- **Timestamp Guard:** The OO can fire callbacks with any timestamp it holds. The guard prevents a delayed T0 callback from falsely triggering a second-dispute escalation after a T1 request is live.
- **Reward Double-Spend Protection:** `_bestEffortRefund` zeros `qd.reward` before the external transfer (CEI). This is a "belt-and-suspenders" insurance against any future re-entry path.
