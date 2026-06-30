# Layer 6 Code Review Fixes — Design Spec

**Issue:** life#22
**Scope:** 4 Minor findings from Layer 6 trust routing code review

## Finding 1 — Test verdict filter misleading

**File:** `app/src/test/java/io/casehub/life/app/service/LifeOutcomeAttestationWriterTest.java`
**Lines:** 63, 98, 196

**Problem:** Filter `.filter(a -> a.verdict != null)` is used to isolate verdict-only attestations, but ALL attestations have non-null verdicts (both verdict-only and dimension attestations set `verdict` in `LifeOutcomeAttestationWriter`). The filter passes everything through.

**Fix:** Change to `.filter(a -> a.trustDimension == null)` — verdict-only attestations have `trustDimension = null`; dimension attestations have `trustDimension != null`.

## Finding 2 — Fully-qualified type in LifeTrustRoutingPolicyProvider

**File:** `app/src/main/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProvider.java`
**Lines:** 59, 62

**Problem:** `io.casehub.platform.api.preferences.PreferenceKey<DoublePreference>` used inline without import.

**Fix:** Add `import io.casehub.platform.api.preferences.PreferenceKey;` and use short form `PreferenceKey<DoublePreference>`.

## Finding 3 — Duplicate trust-routing.yaml

**Files:**
- `app/src/main/resources/casehub/life/trust-routing.yaml`
- `app/src/test/resources/casehub/life/trust-routing.yaml`

**Problem:** Identical files. Test copy is redundant — main resources are on the classpath during `@QuarkusTest`.

**Fix:** Delete `app/src/test/resources/casehub/life/trust-routing.yaml`.

## Finding 4 — ColdStartBehaviorTest incomplete

**File:** `app/src/test/java/io/casehub/life/app/ColdStartBehaviorTest.java`

**Problem:** Tests verify policy availability and CDI injection but don't verify actual cold-start data paths.

**Fix:** Add three tests:
1. `trustGateReturnsEmptyForUnknownActor` — `TrustGateService.currentScore()` returns `Optional.empty()` for a fabricated `life-actor:{uuid}` actorId
2. `trustGateDimensionScoresEmptyForUnknownActor` — `TrustGateService.dimensionScores()` returns empty map
3. `restEndpointReturnsColdStartTrustProfile` — `GET /external-actors/{id}` returns `TrustProfile` with `null` globalScore and empty dimension/capability maps

Replace the weak `trustGateServiceIsAvailable` test (injection-only) with the substantive `trustGateReturnsEmptyForUnknownActor` test which implicitly verifies injection.