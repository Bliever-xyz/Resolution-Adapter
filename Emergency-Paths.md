# Emergency Paths — Architecture Reference

**Protocol:** Bliever Protocol  
**System:** `BlieverUmaAdapter` + `BlieverMarket`  
**Document Scope:** Every emergency and admin-intervention path: flagging, manual resolution, question reset, per-question pause, global pause, and market expiry.

---

## Design Philosophy

The emergency system is built on one fundamental rule: **settlement and claims must never be blockable, under any circumstances.**

Every pause, flag, and freeze applies only to new activity (trading, oracle resolution). The paths that pay traders — `resolve()`, `claim()`, `expireUnresolved()` — are hardened against all admin controls. This asymmetry is intentional: an admin error should never trap winner funds.

---

## Emergency Controls Overview

The following table summarizes the primary emergency controls and their respective scopes.

| Control | Who Sets | What It Blocks | What It Does NOT Block |
| :--- | :--- | :--- | :--- |
| `qd.paused` | `EMERGENCY_ROLE` | Permissionless `adapter.resolve()` | `resolveManually()`, `market.claim()` |
| `qd.manualResolveAt > 0` (flag) | `EMERGENCY_ROLE` | Permissionless `adapter.resolve()` | `resolveManually()` after safety period |
| Global adapter pause | `DEFAULT_ADMIN_ROLE` | `initializeQuestion()`, `adapter.resolve()` | All admin functions, `market.claim()` |
| Market `pause()` | `Factory` | `market.buy()`, `market.sell()` | `market.resolve()`, `market.claim()` |

> **Note:** These four controls are fully **independent**. Setting one never sets another, and clearing one never clears another.

---

## Emergency Decision Tree

The following decision tree guides the appropriate response to various emergency scenarios.

![Emergency Decision Tree](https://private-us-east-1.manuscdn.com/sessionFile/RFp897btO1Cqct7PJ3MyE0/sandbox/o6yjoaGRjQiFrCzLAdzrsW-images_1775923252277_na1fn_L2hvbWUvdWJ1bnR1L2VtZXJnZW5jeV9kZWNpc2lvbl90cmVl.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvUkZwODk3YnRPMUNxY3Q3UEozTXlFMC9zYW5kYm94L282eWpvYUdSalFpRnJDekxBZHpyc1ctaW1hZ2VzXzE3NzU5MjMyNTIyNzdfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVnRaWEpuWlc1amVWOWtaV05wYzJsdmJsOTBjbVZsLnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=t6njHg0vB0W1a93MbbiMq0t8jBoqd5hQ9Y~OA8BnvncRbG4DeTheywduNDFf92c24IB9fI~pS4DlJEICa-y44ZSzGZBICnWqNzhbr-VtoWYc9GGIRDAyCwEXgEVpi6M3PuZD6mnFpJO~KKZYCkQGLu8XvCD6KJlnaXpz8szdXb4tvHvj4vOAzLkak5CI4eS09E45a6CsjtM15gDJqHFrFUV6tAC-4~JgF1muPTB6Jn~y3v~wat5QGcuWkt8CHETolwhDwh5gqDJqs0AWbxWSUMa0AjaQtOrfRnGoHM04Rm~FoFd4B3EK8uvejwJMvNt6ppp~NGzZJgSqC59h9UkiIA__)

---

## Emergency Path 1 — Flag and Manual Resolution

### Use Cases
- UMA DVM returns a demonstrably wrong result (e.g., a governance attack).
- Market ancillary data is fatally ambiguous and the oracle cannot resolve correctly.
- The proposer bot submitted an incorrect outcome and the 2-hour liveness window has already closed.

### Step 1: `flag(questionId)` — EMERGENCY_ROLE
**Effect:** `qd.manualResolveAt = block.timestamp + SAFETY_PERIOD` (1 hour).  
`adapter.resolve()` now reverts with `Flagged`. The optimistic OO path is frozen. The 1-hour safety period exists to give the community time to verify whether the flag is legitimate.

### Step 2a: `unflag(questionId)` — EMERGENCY_ROLE
If the flag was an error, it can be removed **only within the 1-hour safety period**. Once the period expires, the admin cannot remove the flag, preventing manipulation without community observation time.

### Step 2b: `resolveManually(questionId, winningOutcome)` — EMERGENCY_ROLE
After the safety period, the admin can settle the market manually.
1. **Effect:** `qd.resolved = true`
2. **Interaction:** `IBlieverMarket(qd.market).resolve(winningOutcome)`
3. **Refund:** `_bestEffortRefund(questionId, qd)` (never reverts)

---

## Visual State Flow: Flagging & Manual Resolution

The following diagram illustrates the state transitions during a flagging and manual resolution event.

![Emergency State Flow](https://private-us-east-1.manuscdn.com/sessionFile/RFp897btO1Cqct7PJ3MyE0/sandbox/o6yjoaGRjQiFrCzLAdzrsW-images_1775923252277_na1fn_L2hvbWUvdWJ1bnR1L2VtZXJnZW5jeV9zdGF0ZV9mbG93.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvUkZwODk3YnRPMUNxY3Q3UEozTXlFMC9zYW5kYm94L282eWpvYUdSalFpRnJDekxBZHpyc1ctaW1hZ2VzXzE3NzU5MjMyNTIyNzdfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVnRaWEpuWlc1amVWOXpkR0YwWlY5bWJHOTMucG5nIiwiQ29uZGl0aW9uIjp7IkRhdGVMZXNzVGhhbiI6eyJBV1M6RXBvY2hUaW1lIjoxNzk4NzYxNjAwfX19XX0_&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=TaJdPBTq1ebEgPk~LQdV0BZDfTV7K~N8mPCedtG5Ocjm64gR2UUcjAqBudDxv00D1IBxKWSEv5bV~F8IaKgcIJdelnw1lgbx9ce0g~0SFcWH2Otj~Izz2Tmfh-7z1oOLpalcczHHGFkX8S-lpUgeucmYk0hQA9T7FUqR9MgGIiDp1LB1C9XFqlpy1g3L2U1p-vDnyruVcbbnBdBFJopDFAFTtDYBIfrkiZ1zJ3wakrzckSZtpSNU1l2iuESU0HXD2rVpNg4c7qe96iPDrO4lq-kwk0qMA9tIvJNhKYoy4b6OcttXNHJiiilGlsnoQ3oM2BkVOcbVFRCtQqMqvT-rgA__)

---

## Emergency Path 2 — Per-Question Pause

### Use Cases
- Temporary operational halt while investigating a potential oracle issue.
- A suspected DVM manipulation in progress.
- Coordinating with UMA governance before a known bad result settles.

**Effect:** `qd.paused = true`.  
`adapter.resolve()` reverts with `QuestionIsPaused`. `resolveManually()` is NOT blocked. `qd.paused` and `qd.manualResolveAt` are separate fields; unflagging will never silently unpause a question.

---

## Emergency Path 3 — Admin Reset (Failsafe)

### Use Cases
- The `priceDisputed` callback from the OO failed to fire.
- A `TOO_EARLY` resolution happened while `reward > 0`.
- General recovery from any stuck state where the OO request is no longer live.

**Execution:**
1. **Refund:** If reward is sitting on adapter, return it first (hard-reverting `_refund`).
2. **Clear Flag:** If the question was flagged, clear it.
3. **New Request:** Issue a new OO request, paid by `EMERGENCY_ROLE`.

---

## Emergency Path 4 & 5 — Global and Market Pauses

### Global Adapter Pause (`DEFAULT_ADMIN_ROLE`)
Implemented via OpenZeppelin `PausableUpgradeable`. Blocks `initializeQuestion()` and `adapter.resolve()`. It does **not** block emergency functions or `market.claim()`.

### Market Trading Pause (`Factory` only)
Applied to `market.buy()` and `market.sell()`. It does **not** block `market.resolve()`, `market.claim()`, or `market.expireUnresolved()`.

---

## Emergency Path 6 — Market Expiry (Oracle Failure Hatch)

### `market.expireUnresolved()` — Factory only
The safety hatch for complete oracle failure. Can only be called **after** the `resolutionDeadline`.

**Effects:**
- `resolved = true`
- `winningOutcome = outcomeCount` (out-of-bounds sentinel)
- `pool.settleMarket(0)` (zero payout to traders)

---

## `_bestEffortRefund` vs `_refund`

The adapter uses two different refund implementations based on the context:

| Primitive | Context | Behavior |
| :--- | :--- | :--- |
| `_bestEffortRefund` | Settlement-critical paths | Never reverts; uses `try/catch` with raw `IERC20.transfer`. |
| `_refund` | Admin paths only (`reset()`) | Hard-reverting; uses `safeTransfer`. Admin can address root causes before retrying. |

---

## Safety Invariants

- **Manual Resolution:** A flagged question can always be manually resolved; `resolveManually()` does not check `qd.paused`.
- **Single Resolution:** A resolved question cannot be re-resolved. Three independent guards exist across the adapter and market.
- **Safety Period:** The 1-hour safety period cannot be bypassed.
- **Claims:** `market.claim()` has no pause check. Winner payouts are never blocked.
