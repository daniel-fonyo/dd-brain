# GenAI Reranker Score Logging Bug

**Status**: Debug logging deployed to sandbox, awaiting log analysis to confirm root cause
**Date**: 2026-04-01
**feed-service branch**: `dfonyo/debug-missing-genai-embedding-scores` (main checkout, not a worktree)
**Sandbox**: Running with committed debug logging + uncommitted test setup (consumer ID hardcode + Iguazu bypass)

## Background

PR [#62113](https://github.com/doordash/feed-service/pull/62113) added `genai_embedding_similarity_score` to Iguazu `score_modifiers` and `reranker_alpha`/`reranker_beta` to `carousel_details`. The score is read from `generatedRecommendationStoreInfoMap[storeId]?.embeddingScore` during `updateStoreEntity()` in `HomepageProgrammaticCarouselGenerationService`. See `brain/genai-reranker-logging/` for original PR context.

## Problem

After PR #62113 merged, production Snowflake data shows `genai_embedding_similarity_score` is present for only ~4% of cxgen store events (5M out of ~123M). The remaining ~96% have the standard score modifiers but no genai score. The `carousel_details` metadata (alpha, beta, k) IS present on all cxgen events, proving reranking happened.

## Hypothesis (unconfirmed — needs debug log validation)

`generatedRecommendationStoreInfoMap` on `LiteStoreCollection` is populated at `buildCollection()` but empty by the time `updateStoreEntity()` runs for some carousels. Evidence from Snowflake:

- **Correlation between `item_id` and genai score**: Both are read from `generatedRecommendationStoreInfoMap[storeId]` in `updateStoreEntity()`. Stores with genai score (10-mod) also have populated `item_id`. Stores without (6-mod) have empty `item_id`. This suggests the entire map lookup returns null, not individual fields.
- **Per-carousel inconsistency**: Store 395288 has genai score in `cxgen:2` and `cxgen:4` but NOT in `cxgen:0` — same request, same store, different carousels. The map is carousel-specific (lives on `LiteStoreCollection`), so something drops it for certain carousels.

Static analysis of all `copy()` calls in the pipeline shows `generatedRecommendationStoreInfoMap` is preserved (Kotlin data class copy retains unspecified fields). The exact drop point is unknown.

## Debug Logging

Commit `a6b42c5213c` adds `[GENAI_DEBUG]` logging at 4 pipeline stages for `cxgen:` carousels:

| Tag | Location | What it logs |
|---|---|---|
| `buildCollection` | `HomepageGeneratedRecommendationProduct.kt` | Map size/keys at creation, entity IDs |
| `getPostFilterResult_input` | `HomepageGeneratedRecommendationProduct.kt` | Map size before filtering/reranking, missing stores |
| `getPostFilterResult_output` | `HomepageGeneratedRecommendationProduct.kt` | Map size after filtering/reranking, missing stores |
| `postSorting_input` / `postSorting_output` | `HomepageDiscoveryRanker.kt` | Map before/after SDK ranking pipeline, `mapSameRef` check |
| `beforeUpdateStoreEntity` | `HomepageProgrammaticCarouselGenerationService.kt` | Final map state before Iguazu logging, stores not in map |

## Next Step: Analyze Debug Logs

1. Sync debug code to sandbox pod via `devbox run web-group1-remote`
2. Trigger homepage request (browser or curl to sandbox)
3. Pull pod logs: `kubectl logs <pod> -n feed-service-sandbox | grep GENAI_DEBUG`
4. Compare map sizes across stages for the same `collectionId` — find where the map goes from populated to empty
5. Once the drop point is identified: determine root cause and fix

## Snowflake Evidence

Table: `IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE`

**Score modifier distribution for cxgen events** (2026-03-29, hour 16):
| num_score_modifiers | event_count | interpretation |
|---|---|---|
| 6 | 35.9M | Standard set minus sr_multiplier, no genai |
| 7 | 40.6M | Standard 7 modifiers, no genai |
| 9 | 41.4M | Standard + expected_commission + expected_dasher_cost, **NO genai** |
| 10 | 5M | Standard + expected_commission + expected_dasher_cost + **genai_embedding_similarity_score** |

**Per-request analysis** (request `afc3ae7d`):
- Store 395288 in `cxgen:0` → 6 mods (NO genai)
- Store 395288 in `cxgen:2` → **10 mods** (HAS genai)
- Store 395288 in `cxgen:4` → **10 mods** (HAS genai)

## Candidate Fix (pending root cause confirmation)

If the map is truly lost during pipeline traversal (not a data issue from the upstream content systems fetcher), the safest fix is to set `genaiEmbeddingSimilarityScore` directly on entity objects during `getPostFilterResult()` in `HomepageGeneratedRecommendationProduct`, where `data.storeInfoMap` is directly accessible. This avoids relying on the map surviving through the SDK ranking pipeline.

Alternative: find and fix the exact copy/transform that drops the map.

## Key Files

| File | Purpose |
|---|---|
| `HomepageGeneratedRecommendationProduct.kt` | `buildCollection()` + `getPostFilterResult()` — map creation + reranking |
| `HomepageDiscoveryRanker.kt` | `postSorting()` — SDK ranking pipeline post-sort hook |
| `HomepageProgrammaticCarouselGenerationService.kt` | `generate()` → `updateStoreEntity()` loop |
| `HomepageProgrammaticProductServiceUtil.kt:204-228` | `updateStoreEntity()` — reads map, sets genai fields on StoreEntity |
| `GeneratedRecommendationCarouselService.kt:117-221` | `reRankStoresByScoreAndSimilarity()` — alpha^score * beta^similarity |
| `StoreCarouselDataAdapter.kt:2521` | Logging: reads `store.genaiEmbeddingSimilarityScore` for Iguazu |
| `LiteStoreCollection.kt:94` | `generatedRecommendationStoreInfoMap` field definition |

## Score Flow

```
buildCollection() → generatedRecommendationStoreInfoMap = data.storeInfoMap
  ↓
SDK ranking pipeline (rank → score → sort → postSorting) → map preserved via copy()?
  ↓
getPostFilterResult → filter → rerank → deduplicate → copy() preserves map?
  ↓
updateStoreEntity(entity, store, collection)
  → collection.generatedRecommendationStoreInfoMap[productStore.id]?.embeddingScore
  → RETURNS NULL for ~96% of cxgen stores (map is empty for those carousels)
```

## Related

- `brain/genai-reranker-logging/` — Original PR #62113 context (embedding score + alpha/beta logging)
- `brain/genai-combined-score-logging/` — RFC for universal ranking signal logging framework (future fix for this class of problems)
