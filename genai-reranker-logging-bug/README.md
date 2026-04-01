# GenAI Reranker Score Logging Bug

**Status**: Investigation in progress
**Date**: 2026-04-01
**feed-service branch**: master (latest)

## Problem

CxGen (genai) carousels on the homepage have missing `genai_embedding_similarity_score` in Iguazu `score_modifiers`. The `carousel_details` metadata (alpha, beta, k) IS present, proving reranking happened, but per-store embedding scores are null.

## Snowflake Evidence

Table: `IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE`

**Score modifier counts for cxgen events** (2026-03-29, hour 16):
| num_score_modifiers | event_count | interpretation |
|---|---|---|
| 0 | 403K | No scoring at all |
| 6 | 35.9M | Standard set minus sr_multiplier, no genai |
| 7 | 40.6M | Standard 7 modifiers, no genai |
| 9 | 41.4M | Standard + expected_commission + expected_dasher_cost, **NO genai** |
| 10 | 5M | Standard + expected_commission + expected_dasher_cost + **genai_embedding_similarity_score** |

Key finding: ~41M events (9 modifiers) have `expected_commission`/`expected_dasher_cost` but **lack** `genai_embedding_similarity_score`. Only ~5M (10 modifiers) have the genai score. This means both scores are set in the SAME `updateStoreEntity()` call, but the genai score map lookup returns null while the prediction score map lookup succeeds.

**All** cxgen `carousel_details` lack `carousel_rank`, `day_part`, `last_update_date` — confirming they go through the content systems fetcher path (EBR Option 2), where `trackingPayload` is not serialized in the proto.

## Architecture: Two Fetcher Paths

### Decision point
`HomepageGeneratedRecommendationProductService.fetch()` (line 48):
```kotlin
if (DiscoveryExperimentManager.shouldUseGeneratedRecommendationEBROption2(context.experimentMap)) {
    contentSystemsFetcher.fetch(context)  // Content systems path
} else {
    fetcher.fetch(context)  // Legacy path
}
```

### Path 1: Legacy (`GeneratedRecommendationFetcher`)
- Fetches carousels from CRDB via `GeneratedRecommendationWorkflowService`
- Store list fetched separately via `storeListFetchJob`
- Embedding scores: **always non-null** (defaults to 0.0 via `?: 0.0`)
- `trackingPayload` includes `carousel_rank`, `day_part`, `last_update_date`
- Carousel `storeIdList` filtered against store list entities

### Path 2: Content Systems (`GeneratedRecommendationContentSystemsFetcher`)
- Calls content systems gRPC → `GeneratedRecommendationCarouselsPipeline` on server side
- Server creates data via `createGeneratedRecommendationCarousels()` with `RetrievalResult.score: Double` (non-null)
- Serialized to proto `GeneratedRecommendationStoreInfo.embedding_score` (DoubleValue wrapper)
- Client deserializes with `hasEmbeddingScore()` check
- `trackingPayload` NOT in proto → defaults to `emptyMap()` on client
- **In theory, embedding scores should also be non-null** (server always sets them)

## Score Flow (Per-Store)

```
buildCollection()
  → LiteStoreCollection.generatedRecommendationStoreInfoMap = data.storeInfoMap  [Map<Long, GeneratedRecommendationStoreInfo>]

reRankStoresByScoreAndSimilarity()
  → reads embeddingScore ?? 0.0 for computation
  → adds genai_reranker_alpha/beta/k to trackingPayload
  → collection.copy() preserves generatedRecommendationStoreInfoMap

updateStoreEntity(entity, store, collection)
  → genaiEmbeddingSimilarityScore = collection.generatedRecommendationStoreInfoMap[productStore.id]?.embeddingScore
  → expectedDasherCost = collection.storePredictionScoresMap[productStore.id]?.expectedDasherCost

StoreCarouselDataAdapter.generateStoreLogging()
  → store.genaiEmbeddingSimilarityScore?.let { map[GENAI_EMBEDDING_SIMILARITY_SCORE_KEY] = ... }

ContainerEventsGenerator.StoreCarouselGenerator.additionalChildLogging()
  → LoggedValue.GENAI_EMBEDDING_SIMILARITY_SCORE → adds to score_modifiers
```

## Key Files

| File | Purpose |
|---|---|
| `HomepageGeneratedRecommendationProductService.kt` | Fetcher decision (EBR Option 2 DV) |
| `GeneratedRecommendationFetcher.kt` | Legacy fetcher (CRDB + store list) |
| `GeneratedRecommendationContentSystemsFetcher.kt` | Content systems fetcher (gRPC) |
| `GeneratedRecommendationContentSystemsFetcherUtil.kt` | Proto deserialization → `embeddingScore` nullable |
| `GeneratedRecommendationCarouselsPipeline.kt` | Server-side content systems pipeline |
| `GeneratedRecommendationCarouselsSerializer.kt` | Server-side proto serialization |
| `GeneratedRecommendationCarouselService.kt:117-221` | Reranking (alpha^score * beta^similarity) |
| `HomepageProgrammaticCarouselGenerationService.kt:85-171` | Carousel generation + store decoration |
| `HomepageProgrammaticProductServiceUtil.kt:204-228` | `updateStoreEntity()` — sets genai score |
| `StoreCarouselDataAdapter.kt:2521` | Logging: reads `store.genaiEmbeddingSimilarityScore` |
| `ContainerEventsGenerator.kt:456` | Iguazu: `GENAI_EMBEDDING_SIMILARITY_SCORE` in score_modifiers |

## Puzzling Finding

The 9-modifier events have `expected_commission`/`expected_dasher_cost` but NOT `genai_embedding_similarity_score`. BOTH are set in the same `updateStoreEntity()` call:
- `expectedDasherCost = rankInfo?.expectedDasherCost` (from `storePredictionScoresMap`)
- `genaiEmbeddingSimilarityScore = collection.generatedRecommendationStoreInfoMap[productStore.id]?.embeddingScore`

This means `storePredictionScoresMap` has the store but `generatedRecommendationStoreInfoMap` does NOT (or has null `embeddingScore`).

## Hypotheses to Investigate Next

1. **Store ID mismatch**: `productStore.id` type mismatch between `storePredictionScoresMap` (populated during ranking) and `generatedRecommendationStoreInfoMap` (populated during collection building). Both should be `Long` but verify.

2. **Map gets reset during ranking**: The ranking/hydration pipeline might create NEW `LiteStoreCollection` objects that don't carry `generatedRecommendationStoreInfoMap`. Check `BaseDynamicProgrammaticProductService` rank/hydrate steps.

3. **Multiple collection copies**: Between `buildCollection()` and `updateStoreEntity()`, the collection goes through multiple `copy()` calls. While data class `copy()` preserves fields, check if any intermediate step creates a NEW collection from builder (which defaults `generatedRecommendationStoreInfoMap` to `emptyMap()`).

4. **Content systems gRPC proto missing embedding_score for some stores**: The proto field is optional (`google.protobuf.DoubleValue`). If the upstream pipeline (retrieval post-processing) doesn't include scores for some stores, they'd be null on the client.

5. **`storeInfoMapList` vs `storeIdsList` divergence**: The content systems response might have stores in `storeIdsList` that aren't in `storeInfoMapList` (e.g., if the server adds stores during dedup/fallback).

## Next Steps

1. **Add debug logging**: Add temporary logging in `updateStoreEntity()` to log `collection.generatedRecommendationStoreInfoMap.keys` vs `productStore.id` for cxgen carousels
2. **Check ranking pipeline**: Trace how `LiteStoreCollection` flows through `BaseDynamicProgrammaticProductService.rank()` and `rankList()` — does the collection get rebuilt?
3. **Query Snowflake**: Compare store IDs that have vs don't have the genai score within the same request_id + carousel to see if there's a pattern
4. **Check `rankList` / `rankingHydrater`**: These may create new collections that don't carry `generatedRecommendationStoreInfoMap`
