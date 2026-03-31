# Implementation Plan: Intent Prediction Score Logging

## Summary
Add `verticalIntentPredictionScore` (raw model probability) as a new `score_modifier` in Iguazu cross-vertical events. Currently only the derived `vertical_intent_multiplier` is logged.

## Prerequisite: Original Model Integration (PR #58350, merged)
The intent prediction model was integrated end-to-end by Michael Chen + Zhenzhen Liu in PR #58350 (`hp_intent_pred`). That PR:
- Added `VERTICAL_INTENT_PREDICTION` predictor enum + `intentPredictionModel` config field
- Wired Sibyl predictor registration (gated by `enableVerticalIntentPrediction` — config starts with `"intent_model__"`)
- Extracted per-entity predictions in `BaseEntityScorer`, propagated through `ScoreBundle` (9 branches) into `ScoreWithFactors.verticalIntentPredictionScore`
- Applied in `VerticalBlending`: `entityScore *= verticalIntentPredictionScore * intentMultiplier`
- Score is already computed and used in ranking — just not logged to Iguazu events

## Changes Required (9 steps, 8 files + 4 test files)

### Step 1: Add logging constant
**File**: `libraries/domain-util/.../utils/logging/DomainUtilLoggingConstants.kt` (after line 379)
```kotlin
const val VERTICAL_INTENT_PREDICTION_SCORE_KEY = "vertical_intent_prediction_score"
```

### Step 2: Add field to StoreCarousel
**File**: `libraries/discovery-shared/.../model/storecarousel/StoreCarousel.kt` (after `verticalIntentMultiplier`)
```kotlin
val verticalIntentPredictionScore: Double? = null,
```

### Step 3: Add field to ItemCarousel
**File**: `libraries/domain-util/.../carousel/models/itemcarousel/ItemCarousel.kt` (after `verticalIntentMultiplier`)
```kotlin
val verticalIntentPredictionScore: Double? = null,
```

### Step 4: Map ScoreWithFactors → domain models in EntityRankerConfiguration
**File**: `libraries/domain-util/.../ranking/EntityRankerConfiguration.kt`
Add in ALL 4 `.copy()` blocks (after `verticalIntentMultiplier`):
```kotlin
verticalIntentPredictionScore = scoreBundle.storeCarouselScores[it.id]?.verticalIntentPredictionScore,
// or itemCarouselScores for item carousel blocks
```
Sites: ~line 300, ~391, ~682, ~766

### Step 5: Add to StoreCarouselDataAdapterUtil logging
**File**: `libraries/domain-util/.../carousel/utils/StoreCarouselDataAdapterUtil.kt` (after verticalIntentMultiplier block, ~line 586)
```kotlin
storeCarousel.verticalIntentPredictionScore?.let {
    map[VERTICAL_INTENT_PREDICTION_SCORE_KEY] = getSafeNumberValueWithDefault(
        if (!it.isNaN()) { it } else { 0.0 },
    )
}
```

### Step 6: Add to ItemCarouselDataAdapter logging
**File**: `libraries/domain-util/.../facets/adapters/ItemCarouselDataAdapter.kt` (after verticalIntentMultiplier block, ~line 1156)
```kotlin
itemCarousel.verticalIntentPredictionScore?.let {
    map[VERTICAL_INTENT_PREDICTION_SCORE_KEY] = AdapterUtil.getSafeNumberValueWithDefault(it)
}
```

### Step 7: Add to CategoryPreviewUniversalItemCarouselMapper
**File**: `libraries/domain-serialization-mapping/.../CategoryPreviewUniversalItemCarouselMapper.kt` (after verticalIntentMultiplier block, ~line 222)
```kotlin
itemCarousel.verticalIntentPredictionScore?.let {
    put(VERTICAL_INTENT_PREDICTION_SCORE_KEY, AdapterUtil.getSafeNumberValueWithDefault(it))
}
```

### Step 8: Add LoggedValue enum entry in ContainerEventsGenerator
**File**: `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt`

**A.** Add new enum entry (after VERTICAL_INTENT_MULTIPLIER, ~line 2345):
```kotlin
VERTICAL_INTENT_PREDICTION_SCORE(VERTICAL_INTENT_PREDICTION_SCORE_KEY, { b, v ->
    when (b) {
        is Events.CrossVerticalHomePageFeedEvent.Builder ->
            b.addScoreModifiers(
                Events.CrossVerticalHomePageFeedEvent.ScoreModifier.newBuilder()
                    .setName(VERTICAL_INTENT_PREDICTION_SCORE_KEY).setValue(v.numberValue).build(),
            )
        is Events.VerticalPageFeedEvent.Builder ->
            b.addScoreModifiers(
                Events.VerticalPageFeedEvent.ScoreModifier.newBuilder()
                    .setName(VERTICAL_INTENT_PREDICTION_SCORE_KEY).setValue(v.numberValue).build(),
            )
        else -> b
    }
}),
```

**B.** Add to values list (~line 203, after VERTICAL_INTENT_MULTIPLIER):
```kotlin
LoggedValue.VERTICAL_INTENT_PREDICTION_SCORE,
```

**C.** Add import for `VERTICAL_INTENT_PREDICTION_SCORE_KEY`

### Step 9: Update tests (see testing-plan.md)

## Risks
- **No proto changes needed** — `score_modifiers` is repeated `{name, value}`, fully additive
- **No DV gating needed** — logging-only change, zero scoring impact
- **Backward compatible** — downstream consumers filtering by name are unaffected
- **NaN safety** — follows existing pattern with `isNaN()` guard

## Branch
`feat/intent-prediction-score-logging` on feed-service (pushed to origin)
- Commit: `d8fe42189ae` — 12 files, detekt passed, PR review clean
- Remote: https://github.com/doordash/feed-service/tree/feat/intent-prediction-score-logging
