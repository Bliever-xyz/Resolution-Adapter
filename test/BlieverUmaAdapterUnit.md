# BlieverUmaAdapterUnit.t.sol — Test Documentation

## Purpose

Tests every public entry point of `BlieverUmaAdapter` in isolation using a `MockOptimisticOracleV2` and `MockBlieverMarket`. No live UMA deployment is required. All OO interactions are verified via spy call-counters on the mock.

---

## Contract Under Test

`contracts/src/BlieverUmaAdapter.sol` (UUPS proxy)

---

## Test Groups

### 1. `initialize` — Proxy Setup

| Test | What it verifies |
|------|-----------------|
| `test_initialize_rolesAssignedCorrectly` | DEFAULT_ADMIN, FACTORY_ROLE, EMERGENCY_ROLE are assigned to correct addresses |
| `test_initialize_oracleSetCorrectly` | `optimisticOracle` storage matches constructor argument |
| `test_initialize_reverts_zeroOracle` | Reverts when OO address is zero |
| `test_initialize_reverts_zeroAdmin` | Reverts when admin address is zero |
| `test_initialize_reverts_cannotReinitialize` | Proxy cannot be initialized a second time |
| `test_initialize_implementationCannotBeInitialized` | Bare implementation reverts on `initialize()` (`_disableInitializers`) |

---

### 2. `initializeQuestion` — Happy Path

| Test | What it verifies |
|------|-----------------|
| `test_initializeQuestion_binary_happyPath` | Full state written: market, outcomeCount, creator, reward, bond, liveness, oracle snapshot, call counters |
| `test_initializeQuestion_sevenOutcome_happyPath` | 7-outcome market registers correctly |
| `test_initializeQuestion_zeroReward_skipsTransfer` | No token pull when reward = 0; OO still receives request |
| `test_initializeQuestion_zeroBondAndLiveness_skipsThoseCalls` | `setBond` and `setCustomLiveness` NOT called when both are 0 |
| `test_initializeQuestion_emitsQuestionInitializedEvent` | `QuestionInitialized` event emitted with correct fields |
| `test_initializeQuestion_rewardPulledFromFactory` | Factory balance decreases by `reward` amount |

---

### 3. `initializeQuestion` — Revert Guards

| Test | Revert reason |
|------|--------------|
| `test_initializeQuestion_reverts_notFactory` | `AccessControl: missing FACTORY_ROLE` |
| `test_initializeQuestion_reverts_zeroMarket` | `ZeroAddress()` |
| `test_initializeQuestion_reverts_zeroRewardToken` | `ZeroAddress()` |
| `test_initializeQuestion_reverts_emptyAncillaryData` | `InvalidAncillaryData()` |
| `test_initializeQuestion_reverts_ancillaryDataTooLong` | `InvalidAncillaryData()` (> 8086 raw bytes) |
| `test_initializeQuestion_reverts_questionIdMismatch` | `QuestionIdMismatch()` — wrong questionId passed |
| `test_initializeQuestion_reverts_marketQuestionIdMismatch` | `QuestionIdMismatch()` — market.questionId() != computed hash |
| `test_initializeQuestion_reverts_duplicateRegistration` | `AlreadyInitialized()` |
| `test_initializeQuestion_reverts_outcomeCountTooLow` | `InvalidOutcomeCount(1)` |
| `test_initializeQuestion_reverts_outcomeCountTooHigh` | `InvalidOutcomeCount(8)` (> MAX_OUTCOMES = 7) |
| `test_initializeQuestion_reverts_whenGloballyPaused` | `Pausable: paused` |

---

### 4. `resolve` — Happy Path

| Test | What it verifies |
|------|-----------------|
| `test_resolve_binary_outcome0Wins` | Outcome 0 decoded and forwarded to `market.resolve(0)` |
| `test_resolve_binary_outcome1Wins` | Outcome 1 decoded and forwarded to `market.resolve(1)` |
| `test_resolve_sevenOutcome_allOutcomes` | All 7 outcome indices resolve to the correct winner |
| `test_resolve_emitsQuestionResolvedEvent` | `QuestionResolved` emitted with correct `encodedPrice` and `winningOutcome` |
| `test_resolve_withRefund_returnsRewardToCreator` | After DVM escalation (`qd.refund = true`), reward is returned to creator |

---

### 5. `resolve` — Special Sentinel Paths

| Test | What it verifies |
|------|-----------------|
| `test_resolve_tooEarly_noReward_resetsQuestion` | `int256.min` with reward=0 → auto-reset (new OO request, updated `requestTimestamp`) |
| `test_resolve_tooEarly_withReward_flagsForAdmin` | `int256.min` with reward>0 → question flagged (`manualResolveAt` set), `QuestionFlagged` emitted |
| `test_resolve_unresolvable_marksAndEmits` | `int256.max` → `qd.unresolvable = true`, `QuestionUnresolvable` emitted, `ready()` = false |

---

### 6. `resolve` — Revert Guards

| Test | Revert reason |
|------|--------------|
| `test_resolve_reverts_notInitialized` | `NotInitialized()` |
| `test_resolve_reverts_alreadyResolved` | `AlreadyResolved()` |
| `test_resolve_reverts_questionPaused` | `QuestionIsPaused()` |
| `test_resolve_reverts_questionFlagged` | `Flagged()` |
| `test_resolve_reverts_questionUnresolvable` | `Unresolvable()` |
| `test_resolve_reverts_whenGloballyPaused` | `Pausable: paused` |

---

### 7. `flag` / `unflag`

| Test | What it verifies |
|------|-----------------|
| `test_flag_happyPath` | Sets `manualResolveAt = block.timestamp + SAFETY_PERIOD`, emits `QuestionFlagged` |
| `test_unflag_happyPath` | Clears `manualResolveAt = 0`, emits `QuestionUnflagged` |
| `test_flag_reverts_notEmergencyRole` | Access control |
| `test_flag_reverts_notInitialized` | `NotInitialized()` |
| `test_flag_reverts_alreadyFlagged` | `Flagged()` |
| `test_flag_reverts_alreadyResolved` | `AlreadyResolved()` |
| `test_unflag_reverts_notFlagged` | `NotFlagged()` |
| `test_unflag_reverts_safetyPeriodPassed` | `SafetyPeriodPassed()` — cannot unflag after deadline |
| `test_unflag_reverts_notEmergencyRole` | Access control |

---

### 8. `resolveManually`

| Test | What it verifies |
|------|-----------------|
| `test_resolveManually_happyPath_binary` | Resolves market with the correct winner, emits `QuestionManuallyResolved` |
| `test_resolveManually_happyPath_sevenOutcome` | Works for any of the 7 valid outcomes |
| `test_resolveManually_reverts_notEmergencyRole` | Access control |
| `test_resolveManually_reverts_notFlagged` | `NotFlagged()` |
| `test_resolveManually_reverts_safetyPeriodNotPassed` | `SafetyPeriodNotPassed()` — too early |
| `test_resolveManually_reverts_invalidOutcome` | `InvalidOutcome(2, 2)` for binary market |
| `test_resolveManually_reverts_alreadyResolved` | `AlreadyResolved()` |

---

### 9. `pauseQuestion` / `unpauseQuestion`

| Test | What it verifies |
|------|-----------------|
| `test_pauseQuestion_blocksResolve` | `qd.paused = true`, `ready()` = false, `resolve()` reverts |
| `test_unpauseQuestion_restoresResolve` | Clears paused; `resolve()` succeeds again |
| `test_pauseQuestion_emitsEvent` | `QuestionPaused` event |
| `test_unpauseQuestion_emitsEvent` | `QuestionUnpaused` event |
| `test_pauseQuestion_reverts_notEmergency` | Access control |
| `test_pauseQuestion_reverts_notInitialized` | `NotInitialized()` |
| `test_pauseQuestion_reverts_alreadyResolved` | `AlreadyResolved()` |

---

### 10. Global `pause` / `unpause`

| Test | What it verifies |
|------|-----------------|
| `test_globalPause_blocksInitializeQuestion` | `whenNotPaused` guard on `initializeQuestion` |
| `test_globalPause_blocksResolve` | `whenNotPaused` guard on `resolve` |
| `test_globalUnpause_restoresOperation` | Full flow works after unpause |
| `test_globalPause_reverts_notAdmin` | Only `DEFAULT_ADMIN_ROLE` can pause globally |

---

### 11. `updateOptimisticOracle`

| Test | What it verifies |
|------|-----------------|
| `test_updateOptimisticOracle_happyPath` | `optimisticOracle` updated, `OptimisticOracleUpdated` emitted |
| `test_updateOptimisticOracle_existingQuestionResolvesOnOldOO` | Existing question's `_questionOracle` snapshot is unchanged; resolution uses old OO |
| `test_updateOptimisticOracle_newQuestionUsesNewOO` | New question after upgrade gets new OO snapshot |
| `test_updateOptimisticOracle_reverts_zeroAddress` | `ZeroAddress()` |
| `test_updateOptimisticOracle_reverts_notAdmin` | Access control |

---

### 12. Admin `reset()` Failsafe

| Test | What it verifies |
|------|-----------------|
| `test_reset_clearsRefundFlagAndIssuesNewRequest` | Clears `refund` flag; issues new OO request |
| `test_reset_reverts_notEmergencyRole` | Access control |
| `test_reset_reverts_alreadyResolved` | `AlreadyResolved()` |

---

### 13. `ready()` View

| Test | What it verifies |
|------|-----------------|
| `test_ready_returnsFalse_notInitialized` | Not initialized → false |
| `test_ready_returnsFalse_priceNotAvailable` | OO `hasPrice` = false → false |
| `test_ready_returnsTrue_priceAvailable` | All preconditions met → true |
| `test_ready_returnsFalse_paused` | Paused → false |
| `test_ready_returnsFalse_flagged` | Flagged → false |
| `test_ready_returnsFalse_resolved` | Already resolved → false |
| `test_ready_returnsFalse_unresolvable` | Unresolvable → false |

---

### 14. `_tryResetQuestion` Access Guard

| Test | What it verifies |
|------|-----------------|
| `test_tryResetQuestion_reverts_notSelf` | `BlieverUmaAdapter__OnlySelf()` — only `address(this)` may call |

---

### 15. Fuzz Tests

| Test | Property |
|------|----------|
| `testFuzz_resolve_validOutcomeNeverReverts` | For any winner ∈ [0, 2), resolve() succeeds without reverting |

---

## Key Design Decisions

- **MockOptimisticOracleV2** tracks call counts for `requestPrice`, `setEventBased`, `setCallbacks`, `setBond`, `setCustomLiveness` — enabling precise spy assertions.
- **questionId derivation** uses `AncillaryDataLib._appendAncillaryData(factory, rawData)` to reproduce the adapter's exact computation, ensuring tests match production behavior.
- **Oracle upgrade scenario** (test group 11) verifies the two-OO invariant: old questions pin to the old OO, new questions use the new OO.
