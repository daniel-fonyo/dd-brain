# Plan: Log GenAI Combined Score via Generic Score Modifiers

**Date**: 2026-03-25

## Context

The `combinedScore` (`finalScore^alpha * similarity^beta`) computed in `GeneratedRecommendationCarouselService.rerankStoreByBlockReranking()` determines store ordering in GenAI carousels but is **discarded after sorting** (line 186). Only the original `finalScore` is logged as `HORIZONTAL_ELEMENT_SCORE`. This means the actual reranking signal can't be analyzed in Snowflake.

## Design Decisions

### Carry score via `GeneratedRecommendationStoreInfo` (not a new LiteStoreCollection field)
`GeneratedRecommendationStoreInfo` already flows through the pipeline via `LiteStoreCollection.generatedRecommendationStoreInfoMap`. It carries `embeddingScore` today. Adding `combinedScore` to this data class avoids a new field on `LiteStoreCollection`.

### Generic `scoreModifiers` map on StoreEntity (not another typed field)
Instead of adding `val genaiCombinedScore: Double? = null` to `StoreEntity` (which requires explicit handling in the adapter and event generator), add a generic `scoreModifiers: Map<String, Double>` map. The adapter and event generator iterate this map dynamically — no hardcoded keys needed.

Existing typed fields (`genaiEmbeddingSimilarityScore`, `srMultiplier`, etc.) stay as-is. No migration. New scores go into the map going forward.

### What belongs in `score_modifiers` vs `carousel_details`
- **`score_modifiers`** (proto `repeated ScoreModifier`): per-store values that vary across stores within a carousel (e.g. `sr_multiplier`, `genai_combined_score`, `genai_embedding_similarity_score`)
- **`carousel_details`** (trackingPayload): per-carousel constants, same for every store (e.g. `alpha`, `beta`, `carousel_id`)

## Files to Modify (6 production + tests)

| # | File | Change |
|---|------|--------|
| 1 | `GeneratedRecommendationStoreInfo.kt` | Add `combinedScore: Double? = null` |
| 2 | `GeneratedRecommendationCarouselService.kt:206` | Compute combinedScore, update info map entries with it |
| 3 | `StoreEntity.kt` | Add `scoreModifiers: Map<String, Double> = emptyMap()` field + Builder |
| 4 | `HomepageProgrammaticProductServiceUtil.kt:225` | Populate `scoreModifiers` from `info.combinedScore` |
| 5 | `StoreCarouselDataAdapter.kt:2521` | Add generic loop: iterate `store.scoreModifiers`, write all entries to logging map |
| 6 | `ContainerEventsGenerator.kt` | After `LoggedValue.assign()`, iterate remaining score modifier entries from logging map and emit `ScoreModifier` proto for each |

## Step 1: Code Changes

### 1a. GeneratedRecommendationStoreInfo — add field

**File:** `libraries/discovery-utils/.../models/GeneratedRecommendationData.kt`

Add `combinedScore` to the data class:
```kotlin
data class GeneratedRecommendationStoreInfo(
    val storeId: Long,
    val itemId: String?,
    val embeddingScore: Double?,
    val imageUrl: String? = null,
    val combinedScore: Double? = null,
)
```

### 1b. GeneratedRecommendationCarouselService — compute and persist scores

**File:** `libraries/discovery/.../programmatic/GeneratedRecommendationCarouselService.kt`

At line 206, update `generatedRecommendationStoreInfoMap` entries with computed combined scores:
```kotlin
val combinedScoreByStoreId = reRankedStoreScoreMap.associate { it.first.storeId() to it.second }
val updatedStoreInfoMap = collection.generatedRecommendationStoreInfoMap.mapValues { (storeId, info) ->
    info.copy(combinedScore = combinedScoreByStoreId[storeId])
}
return collection.copy(
    entities = combinedStores,
    storeIds = combinedStores.map { it.storeId() },
    trackingPayload = collection.trackingPayload + mapOf(
        "genai_reranker_alpha" to alpha,
        "genai_reranker_beta" to beta,
    ),
    generatedRecommendationStoreInfoMap = updatedStoreInfoMap,
)
```

### 1c. StoreEntity — add generic scoreModifiers map

**File:** `libraries/platform/.../models/StoreEntity.kt`

Add after `genaiEmbeddingSimilarityScore` (line 119):
```kotlin
val scoreModifiers: Map<String, Double> = emptyMap(),
```
Plus corresponding Builder field and `build()` passthrough.

### 1d. HomepageProgrammaticProductServiceUtil — populate scoreModifiers

**File:** `libraries/discovery/.../programmatic/util/HomepageProgrammaticProductServiceUtil.kt`

In `updateStoreEntity()`, after line 225, build `scoreModifiers` from the info map:
```kotlin
scoreModifiers = listOfNotNull(
    collection.generatedRecommendationStoreInfoMap[productStore.id]?.combinedScore
        ?.let { "genai_combined_score" to it },
).toMap(),
```

### 1e. StoreCarouselDataAdapter — generic loop for scoreModifiers

**File:** `libraries/domain-util/.../facets/adapters/StoreCarouselDataAdapter.kt`

After existing explicit score blocks (line ~2521), add a single loop:
```kotlin
store.scoreModifiers.forEach { (key, value) ->
    map[key] = getSafeNumberValueWithDefault(value)
}
```

### 1f. ContainerEventsGenerator — generic ScoreModifier emission

**File:** `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt`

After `LoggedValue.assign(storeLogging, this, STORE_LOGGED_VALUES)`, add a generic pass that emits `ScoreModifier` for any numeric entries in the logging map that weren't already handled by a `LoggedValue`:
```kotlin
// Emit all dynamic score modifiers not handled by LoggedValue
val handledKeys = STORE_LOGGED_VALUES.map { it.key }.toSet()
storeLogging.forEach { (key, value) ->
    if (key !in handledKeys && value.hasNumberValue()) {
        when (this) {
            is Events.CrossVerticalHomePageFeedEvent.Builder ->
                addScoreModifiers(
                    Events.CrossVerticalHomePageFeedEvent.ScoreModifier.newBuilder()
                        .setName(key).setValue(value.numberValue).build(),
                )
        }
    }
}
```

No new `LoggedValue` enum entry needed. No new constants file entry needed.

## Step 2: Unit Tests

### 2a. GeneratedRecommendationCarouselServiceTest
Verify that after `reRankStoresByScoreAndSimilarity()`, the returned collection's `generatedRecommendationStoreInfoMap` entries have correct `combinedScore` values.

### 2b. StoreCarouselDataAdapterTest
Verify that a `StoreEntity` with `scoreModifiers = mapOf("genai_combined_score" to 0.75)` produces a logging map containing `"genai_combined_score"` → `0.75`.

### 2c. ContainerEventsGeneratorTest
Verify that numeric entries in the store logging map that aren't in `STORE_LOGGED_VALUES` are emitted as `ScoreModifier` proto entries.

## Step 3: Run Unit Tests
```bash
./gradlew :libraries:discovery:test --tests "*GeneratedRecommendationCarouselServiceTest*"
./gradlew :libraries:domain-util:test --tests "*StoreCarouselDataAdapterTest*"
./gradlew :libraries:domain-util:test --tests "*ContainerEventsGeneratorTest*"
```

## Step 4: Sandbox Verification (optional)
- `/sandbox-setup` → deploy local code
- `/sandbox-test` → load homepage, extract carousels
- `/validate-iguazu` → query Snowflake for `genai_combined_score` in `SCORE_MODIFIERS`

## Step 5: PR
- Title: `Add genai_combined_score via generic score modifier pipeline`
- Description: summary of generic approach, test plan, Snowflake validation

## Future Scores
After this PR, adding a new per-store score modifier requires only:
1. Put the value into `scoreModifiers` at the source (1-2 files)

No adapter, event generator, constants, or `LoggedValue` changes needed.
