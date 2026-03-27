# Plan 2: Log GenAI Combined Score via RankingSignalCollector

**Date**: 2026-03-27 (revised)
**Depends on**: Plan 1 (infrastructure)
**Design doc**: [design.md](design.md)

## Goal

Log `combinedScore` (`finalScore^alpha * similarity^beta`) to Snowflake as `genai_combined_score` using the RankingSignalCollector infrastructure from Plan 1. Also log the ranking model name as a string signal.

## Files to Modify (1 production + tests)

| # | File | Change |
|---|------|--------|
| 1 | `GeneratedRecommendationCarouselService.kt` | Add `collector.record()` calls after computing combinedScore |

**That's it.** No new data class fields, no adapter changes, no event generator changes. The infrastructure from Plan 1 handles everything downstream.

## Code Change

### GeneratedRecommendationCarouselService — record signals

**File**: `libraries/discovery/.../programmatic/GeneratedRecommendationCarouselService.kt`

In `rerankStoreByBlockReranking()`, after computing combined scores and before returning (around line 206):

```kotlin
// Record ranking signals for Snowflake logging
val collector = context.rankingSignalCollector

// Per-entity: combined score for each reranked store
reRankedStoreScoreMap.forEach { (store, combinedScore) ->
    collector.record(collection.id, store.storeId(), "genai_combined_score", combinedScore)
}

// Per-carousel: reranker parameters
collector.record(collection.id, "genai_reranker_alpha", alpha)
collector.record(collection.id, "genai_reranker_beta", beta)
collector.record(collection.id, "genai_ranking_model", "block_reranking")

return collection.copy(
    entities = combinedStores,
    storeIds = combinedStores.map { it.storeId() },
    // trackingPayload no longer needs alpha/beta — they go through the collector now
)
```

**Note**: `context` here is the `ExploreContext` passed through the pipeline. If it's not directly available in this method, it may need to be threaded from the caller (`HomepageGeneratedRecommendationProduct.getPostFilterResult()`).

### What about embeddingScore?

The `embeddingScore` (similarity) is already available in `GeneratedRecommendationStoreInfo.embeddingScore`. We can optionally log it through the collector too for completeness:

```kotlin
collection.generatedRecommendationStoreInfoMap.forEach { (storeId, info) ->
    info.embeddingScore?.let {
        collector.record(collection.id, storeId, "genai_embedding_score", it)
    }
}
```

This is optional — PR #62113 may already log this via a typed field. Decide based on whether that PR is merged.

## What Happens Downstream (automatic via Plan 1)

1. `StoreCarouselDataAdapter` calls `RankingSignalWriter.writeEntitySignals()` → `genai_combined_score` and `genai_ranking_model` appear in the store's logging map.
2. `RankingSignalWriter.writeCarouselSignals()` → `genai_reranker_alpha`, `genai_reranker_beta`, `genai_ranking_model` appear in the logging map.
3. `ContainerEventsGenerator` forwards unhandled keys → they land in `ranking_signals` proto map.
4. Iguazu writes to Snowflake → `RANKING_SIGNALS` column populated.

No code changes needed for any of these steps.

## Snowflake Query

```sql
SELECT
    RANKING_SIGNALS:genai_combined_score::DOUBLE AS genai_combined_score,
    RANKING_SIGNALS:genai_reranker_alpha::DOUBLE AS alpha,
    RANKING_SIGNALS:genai_reranker_beta::DOUBLE AS beta,
    RANKING_SIGNALS:genai_ranking_model::VARCHAR AS ranking_model,
    STORE_ID,
    FACET_NAME
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
WHERE IGUAZU_PARTITION_DATE = CURRENT_DATE()
  AND IGUAZU_PARTITION_HOUR = HOUR(CURRENT_TIMESTAMP()) - 1
  AND RANKING_SIGNALS:genai_combined_score IS NOT NULL
```

## Unit Tests

### GeneratedRecommendationCarouselServiceTest
- After `reRankStoresByScoreAndSimilarity()`, verify `context.rankingSignalCollector.forEntity(collectionId, storeId)` contains `"genai_combined_score"` with correct value matching `finalScore^alpha * similarity^beta`.
- Verify `context.rankingSignalCollector.forCarousel(collectionId)` contains `"genai_reranker_alpha"`, `"genai_reranker_beta"`, `"genai_ranking_model"`.

## Run Tests
```bash
./gradlew :libraries:discovery:test --tests "*GeneratedRecommendationCarouselServiceTest*"
```

## Sandbox Verification
- `/sandbox-setup` → deploy local code (Plan 1 + Plan 2)
- `/sandbox-test` → load homepage, trigger GenAI carousels
- `/validate-iguazu` → query Snowflake for `genai_combined_score` in `RANKING_SIGNALS`

## PR
- **Title**: `Log genai_combined_score via RankingSignalCollector`
- **Description**: Records combined score, alpha, beta, and ranking model into the ranking signal collector during GenAI carousel reranking. Automatically flows to Snowflake `RANKING_SIGNALS` column via Plan 1 infrastructure. One file changed, zero downstream wiring.
