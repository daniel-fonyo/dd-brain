# Plan 2: Log genai_combined_score via Score Modifier Pipeline

**Date**: 2026-03-25
**Depends on**: Plan 1 (infrastructure)

## Goal

Log `combinedScore` (`finalScore^alpha * similarity^beta`) to Snowflake `SCORE_MODIFIERS` column as `genai_combined_score`. Uses the generic `scoreModifiers` pipeline from Plan 1.

## Files to Modify (3 production + tests)

| # | File | Change |
|---|------|--------|
| 1 | `GeneratedRecommendationStoreInfo.kt` | Add `combinedScore: Double? = null` |
| 2 | `GeneratedRecommendationCarouselService.kt` | Compute combinedScore, update info map entries |
| 3 | `HomepageProgrammaticProductServiceUtil.kt` | Populate `scoreModifiers` from `info.combinedScore` |

## Code Changes

### 1. GeneratedRecommendationStoreInfo — add field

**File:** `libraries/discovery-utils/.../models/GeneratedRecommendationData.kt`

```kotlin
data class GeneratedRecommendationStoreInfo(
    val storeId: Long,
    val itemId: String?,
    val embeddingScore: Double?,
    val imageUrl: String? = null,
    val combinedScore: Double? = null,
)
```

### 2. GeneratedRecommendationCarouselService — compute and persist

**File:** `libraries/discovery/.../programmatic/GeneratedRecommendationCarouselService.kt`

At line 206, update info map entries with computed combined scores:
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

### 3. HomepageProgrammaticProductServiceUtil — populate scoreModifiers

**File:** `libraries/discovery/.../programmatic/util/HomepageProgrammaticProductServiceUtil.kt`

In `updateStoreEntity()`, after line 225:
```kotlin
scoreModifiers = listOfNotNull(
    collection.generatedRecommendationStoreInfoMap[productStore.id]?.combinedScore
        ?.let { "genai_combined_score" to it },
).toMap(),
```

This is the only code needed — the adapter and event generator handle it automatically via Plan 1's generic loop.

## Unit Tests

### GeneratedRecommendationCarouselServiceTest
- After `reRankStoresByScoreAndSimilarity()`, returned collection's `generatedRecommendationStoreInfoMap` entries have correct `combinedScore` values matching `finalScore^alpha * similarity^beta`.

### HomepageProgrammaticProductServiceUtilTest
- When `generatedRecommendationStoreInfoMap` has `combinedScore = 0.85` for a store, the decorated `StoreEntity.scoreModifiers` contains `"genai_combined_score" → 0.85`.

## Run Tests
```bash
./gradlew :libraries:discovery:test --tests "*GeneratedRecommendationCarouselServiceTest*"
./gradlew :libraries:discovery:test --tests "*HomepageProgrammaticProductServiceUtilTest*"
```

## Sandbox Verification
- `/sandbox-setup` → deploy local code (Plan 1 + Plan 2)
- `/sandbox-test` → load homepage, trigger GenAI carousels
- `/validate-iguazu` → query Snowflake for `genai_combined_score` in `SCORE_MODIFIERS`

## PR
- Title: `Log genai_combined_score to Iguazu score modifiers`
- Description: Adds `combinedScore` to `GeneratedRecommendationStoreInfo`, computes it during reranking, and populates `StoreEntity.scoreModifiers`. Automatically emitted to Snowflake via the generic score modifier pipeline.
