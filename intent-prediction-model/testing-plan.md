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

## Part 2: E2E Sandbox Tests

### Prerequisites
1. Code deployed to sandbox via `devbox run web-group1-remote`
2. Iguazu bypass applied (`val isSandboxEnv = false`)
3. Consumer ID hardcoded (`return 757606047L`)
4. `enableVerticalIntentPrediction` experiment flag active for test consumer

### Flow
1. Browse homepage at `https://www.doordashtest.com/` with test credentials
2. Record UTC timestamp of page load
3. Check pod logs for immediate validation:
   ```bash
   kubectl logs <pod> --since=5m | grep "vertical_intent_prediction_score"
   ```
4. Query Snowflake for detailed validation (use `/validate-iguazu`)

### Key Snowflake Queries

**Verify presence in score_modifiers:**
```sql
SELECT e.FACET_ID, sm.value:name::STRING AS name, sm.value:value::DOUBLE AS val
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE e,
    LATERAL FLATTEN(input => PARSE_JSON(e.SCORE_MODIFIERS)) sm
WHERE e.IGUAZU_PARTITION_DATE = '<date>'
  AND e.IGUAZU_PARTITION_HOUR BETWEEN <h-1> AND <h+1>
  AND e.CONSUMER_ID = 757606047
  AND sm.value:name::STRING = 'vertical_intent_prediction_score'
```

**Cross-reference with existing multiplier:**
```sql
SELECT e.FACET_ID,
    MAX(CASE WHEN sm.value:name::STRING = 'vertical_intent_prediction_score' THEN sm.value:value::DOUBLE END) AS raw_score,
    MAX(CASE WHEN sm.value:name::STRING = 'vertical_intent_multiplier' THEN sm.value:value::DOUBLE END) AS multiplier
FROM ... LATERAL FLATTEN(...) sm
WHERE ... AND sm.value:name::STRING IN ('vertical_intent_prediction_score', 'vertical_intent_multiplier')
GROUP BY e.FACET_ID
```

### Acceptance Criteria
| Check | Pass Condition |
|---|---|
| Pod logs show field | At least one event contains it |
| Snowflake events have modifier | Coverage > 0% (experiment-dependent) |
| Values in valid range | All values >= 0, no NaN/Infinity |
| Null when experiment off | 0% rows have field when model disabled |
| Coexists with multiplier | Both present on same carousels |
| No regression on existing modifiers | All prior modifiers maintain coverage |
