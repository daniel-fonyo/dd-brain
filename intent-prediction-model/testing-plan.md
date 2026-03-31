# Testing Plan: Intent Prediction Score Logging

## Part 1: Unit Tests

### ContainerEventsGeneratorTest (CrossVertical + VerticalPage events)
**File**: `libraries/domain-util/src/test/kotlin/.../iguazu/ContainerEventsGeneratorTest.kt`

| Test | What | How |
|------|------|-----|
| Score present (CrossVertical) | Add `VERTICAL_INTENT_PREDICTION_SCORE_KEY=0.72` to extraLoggingFields | Assert `scoreModifiersList.find { it.name == KEY }?.value == 0.72` |
| Score absent (CrossVertical) | Don't set the key | Assert `scoreModifiersList.none { it.name == KEY }` |
| Score present (VerticalPage) | Same pattern, tile carousel facet | Assert modifier present with correct value |

### StoreCarouselDataAdapterUtilTest
**File**: `libraries/domain-util/src/test/kotlin/.../utils/StoreCarouselDataAdapterUtilTest.kt`

| Test | Input | Expected |
|------|-------|----------|
| Present | `verticalIntentPredictionScore = 0.72` | `fieldsMap[KEY]?.numberValue == 0.72` |
| NaN → 0.0 | `verticalIntentPredictionScore = Double.NaN` | `fieldsMap[KEY]?.numberValue == 0.0` |
| Null → absent | field not set | `fieldsMap.doesNotContainKey(KEY)` |

### ItemCarouselDataAdapterTest
**File**: `libraries/domain-util/src/test/kotlin/.../facets/adapters/ItemCarouselDataAdapterTest.kt`

| Test | Input | Expected |
|------|-------|----------|
| Present | set on itemCarousel | `facet.logging.fieldsMap[KEY]?.numberValue == expected` |
| Null → absent | field not set | `doesNotContainKey(KEY)` |

### CategoryPreviewUniversalItemCarouselMapperTest
**File**: `libraries/domain-serialization-mapping/src/test/kotlin/.../CategoryPreviewUniversalItemCarouselMapperTest.kt`

| Test | Expected |
|------|----------|
| Present | `fields[KEY]?.numberValue == expected` |
| Absent | `fields.containsKey(KEY) == false` |

### Edge Cases
- Score of exactly 0.0 → should be PRESENT (not omitted)
- Score of exactly 1.0 → value matches
- Very small score (1e-10) → no precision loss

### Run Command
```bash
cd /Users/daniel.fonyo/Projects/feed-service
./gradlew :libraries:domain-util:test --tests "*ContainerEventsGeneratorTest*"
./gradlew :libraries:domain-util:test --tests "*StoreCarouselDataAdapterUtilTest*"
./gradlew :libraries:domain-util:test --tests "*ItemCarouselDataAdapterTest*"
./gradlew :libraries:domain-serialization-mapping:test --tests "*CategoryPreviewUniversalItemCarouselMapperTest*"
```

## Part 2: PR Review Self-Reinforcement Loop

Run the PR review skill against the diff (no GitHub PR needed) to catch issues before commit.

### Process
1. **Review agent** runs PR review dimensions against `git diff main...HEAD` in the worktree
2. Review agent outputs comments tagged `[critical]`, `[bug]`, `[suggestion]`, `[nit]` with file:line references
3. **Fix agent** receives the review comments and applies fixes in the worktree
4. Re-run review agent to verify all `[critical]` and `[bug]` items are resolved
5. Repeat until clean (max 2 iterations)

### Review Dimensions
- **Correctness**: Does the logging key match? Are all 4 EntityRankerConfiguration copy blocks covered?
- **Design**: Does the new field follow existing patterns exactly?
- **Performance**: No new allocations, no hot-path changes
- **Readability**: Naming consistent with adjacent fields
- **Testing**: Are all test cases covered (present/absent/NaN)?
- **Kotlin style**: Follows Google Android Kotlin Style Guide

### Acceptance
- Zero `[critical]` or `[bug]` comments remaining
- All `[suggestion]` items either addressed or explicitly deferred with rationale

## Part 3: E2E Sandbox Tests

### Sandbox Test State Setup (local-only, NEVER committed)

The intent prediction model is gated by DV flags and runtime config. For sandbox testing, ALL gates must be bypassed to exercise the full code path. A test showing null/empty for the score is NOT a pass.

#### Required Local Hacks (4 files)

| # | File | Change | Why |
|---|------|--------|-----|
| 1 | `pipelines/homepage/.../HomepageRequestToContext.kt` | `getConsumerIdFromRequest` → `return 757606047L` | Pin to known test consumer |
| 2 | `libraries/platform/.../iguazu/IguazuModule.kt` | Comment out entire gating block in `sendEvent()` (all early-return guards before `try`) | Bypass sandbox Iguazu gating so events flow to Snowflake |
| 3 | `libraries/sdk-p13n/.../scoring/SibylRegressor.kt` | Replace the `verticalIntentPredictorName` block (lines 276-289): bypass `enableVerticalIntentPrediction` DV check and `config.intentPredictionModel` config check. Hardcode model ID `1774285074` directly. | The DV `hp_vertical_blending_config` must start with `"intent_model__"` to enable the model. This DV is not set in sandbox. |
| 4 | `libraries/domain-util/.../ranking/VerticalBlending.kt` | In `scoreEntity()`, remove `config.intentPredictionModel.isNotEmpty()` from the if-condition (line 308), keep only `blendableEntity.verticalIntentPredictionScore != null` | The blending code has a second gate on the config — without this, even if the model returns a score, it won't be multiplied in |

#### SibylRegressor.kt hack (replace lines 276-289):
```kotlin
// SANDBOX HACK: bypass DV gate and hardcode model ID for testing
if (context.additionalPredictors?.verticalIntentPredictorName != null) {
    val verticalIntentPredictor = Predictor.newBuilder()
        .setPredictorName(context.additionalPredictors.verticalIntentPredictorName.label)
        .setModelOverrideId("1774285074")
        .build()
    predictors.add(verticalIntentPredictor)
}
```

#### VerticalBlending.kt hack (line 307-310):
```kotlin
// SANDBOX HACK: bypass config check, only check if score exists
if (
    blendableEntity.verticalIntentPredictionScore != null
) {
```

#### Debug Logging (optional but recommended)
Add these temporary log lines for pipeline visibility:

| Tag | File | What it shows |
|-----|------|---------------|
| `DEBUG_IGUAZU_EMIT` | `IguazuModule.kt` (in `try` block) | Topic, event count at emission point |
| `DEBUG_INTENT_SCORE` | `VerticalBlending.kt` (after scoring) | Per-entity `verticalIntentPredictionScore`, `intentMultiplier`, `entityScore` |
| `DEBUG_SCORE_MODIFIERS` | `ContainerEventsGenerator.kt` (after `LoggedValue.assign`) | All score_modifier values from logging map |

#### Verification Before Browsing
After applying hacks and syncing (`devbox run`), verify the build succeeds. Then:
```bash
kubectl -n feed-service-sandbox logs <pod> --since=2m | grep "DEBUG_" | head -5
```
If no DEBUG_ lines appear, the hacks weren't compiled in — check for build errors.

### Prerequisites
1. Code deployed to sandbox via `devbox run web-group1-remote` (from feed-service main checkout on the feature branch)
2. All 4 local hacks applied (consumer ID + Iguazu bypass + DV bypass + blending bypass)
3. Build succeeds (252 tasks, 0 failures)
4. Pod is running and service is ready

### Flow
1. Browse homepage at `https://www.doordashtest.com/` with test credentials
2. Record UTC timestamp of page load
3. Check pod logs for immediate validation:
   ```bash
   kubectl -n feed-service-sandbox logs <pod> --since=5m | grep "DEBUG_INTENT_SCORE" | head -10
   ```
   **MUST see `verticalIntentPredictionScore=<non-null value>`** — if null, the model call is still gated.
4. Check Iguazu emission:
   ```bash
   kubectl -n feed-service-sandbox logs <pod> --since=5m | grep "DEBUG_IGUAZU_EMIT" | head -10
   ```
5. Query Snowflake for detailed validation (after 3-5 min propagation delay)

### Key Snowflake Queries

**Check events exist:**
```sql
SELECT COUNT(*) AS cnt
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
WHERE IGUAZU_PARTITION_DATE = '<date>'
  AND IGUAZU_PARTITION_HOUR BETWEEN <h-1> AND <h+1>
  AND CONSUMER_ID = 757606047;
```

**Verify vertical_intent_prediction_score in score_modifiers:**
```sql
SELECT e.FACET_ID, sm.value:name::STRING AS name, sm.value:value::DOUBLE AS val
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE e,
    LATERAL FLATTEN(input => PARSE_JSON(e.SCORE_MODIFIERS)) sm
WHERE e.IGUAZU_PARTITION_DATE = '<date>'
  AND e.IGUAZU_PARTITION_HOUR BETWEEN <h-1> AND <h+1>
  AND e.CONSUMER_ID = 757606047
  AND sm.value:name::STRING = 'vertical_intent_prediction_score'
LIMIT 20;
```

**All score_modifiers summary:**
```sql
SELECT sm.value:name::STRING AS modifier_name, COUNT(*) AS cnt,
    AVG(sm.value:value::DOUBLE) AS avg_val,
    MIN(sm.value:value::DOUBLE) AS min_val,
    MAX(sm.value:value::DOUBLE) AS max_val
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE e,
    LATERAL FLATTEN(input => PARSE_JSON(e.SCORE_MODIFIERS)) sm
WHERE e.IGUAZU_PARTITION_DATE = '<date>'
  AND e.IGUAZU_PARTITION_HOUR BETWEEN <h-1> AND <h+1>
  AND e.CONSUMER_ID = 757606047
GROUP BY modifier_name
ORDER BY cnt DESC;
```

**Cross-reference raw score with multiplier:**
```sql
SELECT e.FACET_ID,
    MAX(CASE WHEN sm.value:name::STRING = 'vertical_intent_prediction_score' THEN sm.value:value::DOUBLE END) AS raw_score,
    MAX(CASE WHEN sm.value:name::STRING = 'vertical_intent_multiplier' THEN sm.value:value::DOUBLE END) AS multiplier
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE e,
    LATERAL FLATTEN(input => PARSE_JSON(e.SCORE_MODIFIERS)) sm
WHERE e.IGUAZU_PARTITION_DATE = '<date>'
  AND e.IGUAZU_PARTITION_HOUR BETWEEN <h-1> AND <h+1>
  AND e.CONSUMER_ID = 757606047
  AND sm.value:name::STRING IN ('vertical_intent_prediction_score', 'vertical_intent_multiplier')
GROUP BY e.FACET_ID
LIMIT 20;
```

**Fallback consumer ID** (if hardcode didn't take effect):
Replace `CONSUMER_ID = 757606047` with `CONSUMER_ID = 1125900615344792` (test account's real ID).

### Acceptance Criteria
| Check | Pass Condition |
|---|---|
| DEBUG_INTENT_SCORE shows non-null scores | `verticalIntentPredictionScore` is a real number (e.g., ~15.67) |
| DEBUG_IGUAZU_EMIT shows events | `home_page_store_ranker_compact_event` topic emitted |
| Snowflake has `vertical_intent_prediction_score` | At least 1 row with the modifier present |
| Values are real numbers | No NaN, no Infinity, no 0.0 placeholders |
| Coexists with `vertical_intent_multiplier` | Both present on same facets |
| No errors in pipeline | Zero errors related to Sibyl, model 1774285074, or intent prediction |
| No regression on existing modifiers | `calibration_multiplier`, `vertical_boost_weight`, etc. all still present |
