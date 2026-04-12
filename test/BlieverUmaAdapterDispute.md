# BlieverUmaAdapterDispute.t.sol — Test Documentation

## Purpose

Tests the `priceDisputed` callback and the full dispute-escalation lifecycle of `BlieverUmaAdapter`. This is the most complex part of the oracle lifecycle: a dispute triggers either a reset (first dispute) or DVM escalation (second dispute), and various guard conditions must all hold correctly.

---

## Contract Under Test

`contracts/src/BlieverUmaAdapter.sol` — specifically:
- `priceDisputed()` (IOptimisticRequester callback)
- `priceProposed()` / `priceSettled()` (no-ops)
- `_tryResetQuestion()` / `_resetQuestion()` (internal, exercised indirectly)

---

## Dispute Lifecycle (Adapter State Machine)

```
initializeQuestion
        │
        ▼
   [requestTimestamp T0]
        │
  ── First Dispute ──────────────────────────────────────────────
        │ priceDisputed(T0)
        ▼
   qd.reset = true
   qd.requestTimestamp = T1 (block.timestamp of dispute)
   New OO.requestPrice issued (T1)
        │
  ── Second Dispute ─────────────────────────────────────────────
        │ priceDisputed(T1)
        ▼
   qd.refund = true
   QuestionEscalatedToDVM emitted
   No new OO request (DVM handles it)
        │
  ── resolve() ──────────────────────────────────────────────────
        │ OO.settleAndGetPrice returns valid price
        ▼
   market.resolve(winner)
   _bestEffortRefund (reward returned to creator)
   QuestionResolved emitted
```

---

## Test Groups

### 1. First Dispute → Question Reset

| Test | What it verifies |
|------|-----------------|
| `test_priceDisputed_firstDispute_resetsQuestion` | `qd.reset = true`, `qd.requestTimestamp` updated to current block, new OO request issued, `QuestionReset` emitted |
| `test_priceDisputed_firstDispute_questionOracleUpdated` | After reset, `_questionOracle` snapshot updated to the CURRENT global OO (important for mid-flight upgrades) |

---

### 2. Second Dispute → DVM Escalation

| Test | What it verifies |
|------|-----------------|
| `test_priceDisputed_secondDispute_escalatesToDVM` | `qd.refund = true`, `QuestionEscalatedToDVM` emitted |
| `test_priceDisputed_secondDispute_noNewOORequest` | Second dispute does NOT issue another OO request (DVM handles settlement) |

---

### 3. Already-Resolved Question → Reward Refund on Callback

| Test | What it verifies |
|------|-----------------|
| `test_priceDisputed_alreadyResolved_returnsReward` | When OO callback fires after manual resolution: adapter does not revert, does not corrupt state |

*Rationale:* The OO can fire callbacks asynchronously after `resolveManually()`. The adapter must be resilient — a revert here would block the disputer's OO transaction.

---

### 4. Stale Timestamp → Callback Silently Ignored

| Test | What it verifies |
|------|-----------------|
| `test_priceDisputed_staleTimestamp_silentlyIgnored` | If `timestamp != qd.requestTimestamp`, no state mutation occurs (no reset, no refund flag, no new OO request) |

*Rationale:* After a question reset, the old OO may still fire callbacks with the stale `T0` timestamp. These must be silently discarded to prevent double-reset.

---

### 5. Reset Failure → Question Flagged

| Test | What it verifies |
|------|-----------------|
| `test_priceDisputed_resetFails_questionFlagged` | When `_tryResetQuestion` fails (OO revert via `requestShouldRevert = true`): `qd.manualResolveAt` is set, `QuestionFlagged` emitted — the disputer's tx still lands |

*Rationale:* The `try/catch` wrapper ensures that an OO-side failure never reverts the disputer's bond submission.

---

### 6. `priceProposed` / `priceSettled` Are No-Ops

| Test | What it verifies |
|------|-----------------|
| `test_priceProposed_isNoOp_doesNotRevert` | Calling `priceProposed` from the known oracle does not change state or revert |
| `test_priceSettled_isNoOp_doesNotRevert` | Calling `priceSettled` does not resolve the question or change state |

---

### 7. `onlyOptimisticOracle` Guard

| Test | What it verifies |
|------|-----------------|
| `test_priceDisputed_reverts_notKnownOracle` | Call from an unknown address reverts with `NotOptimisticOracle()` |
| `test_priceDisputed_acceptsOldOracleAfterUpgrade` | OLD oracle address (registered in `_knownOracles`) is still accepted after `updateOptimisticOracle` |

*Rationale:* Disputes from the old OO must continue to be accepted for questions initialized before the upgrade.

---

### 8. Post-Reset Re-Resolution

| Test | What it verifies |
|------|-----------------|
| `test_resolveAfterReset_succeedsOnNewTimestamp` | After first dispute reset, `resolve()` works using the updated `requestTimestamp` |

---

### 9. DVM Path End-to-End

| Test | What it verifies |
|------|-----------------|
| `test_resolveAfterDVMEscalation_succeedsWithRefundFlag` | Two disputes → DVM → `resolve()` succeeds; `qd.reward` zeroed; creator receives refund |

---

### 10. Fuzz Tests

| Test | Property |
|------|----------|
| `testFuzz_priceDisputed_staleTimestampNeverMutatesState` | For any `badTimestamp != requestTimestamp`, the priceDisputed callback must never mutate question state |

---

## Key Design Decisions

- **`_simulateDispute` helper** calls `priceDisputed` from `address(mockOO)` using the stored `qd.ancillaryData` and `qd.requestTimestamp`. This exactly mirrors how the real OO fires the callback.
- **Reset failure testing** is achieved via `MockOptimisticOracleV2.setShouldRevert(true)`, which causes `_requestPrice → OO.requestPrice()` to revert inside the `try` block.
- **DVM path** is tested by firing two consecutive disputes, which advances `qd.refund = true`, then resolving normally via `resolve()` to verify the reward refund flow completes.
