# GenAI Reranker Score Logging Bug

**Status**: Root cause narrowed to map loss in pipeline, fix + debug logging deployed
**Date**: 2026-04-01
**feed-service branch**: `fix/genai-reranker-score-logging` (worktree at `.claude/worktrees/fix-genai-reranker-score-logging`)

## Problem

CxGen (genai) carousels on the homepage have missing `genai_embedding_similarity_score` in Iguazu `score_modifiers`. The `carousel_details` metadata (alpha, beta, k) IS present, proving reranking happened, but per-store embedding scores are null.

## Root Cause

`generatedRecommendationStoreInfoMap` on `LiteStoreCollection` is set during `buildCollection()` but is empty by the time `updateStoreEntity()` runs in `HomepageProgrammaticCarouselGenerationService`. All `copy()` calls in the ranking pipeline were verified to preserve the field — the exact drop point hasn't been identified via static analysis alone. Debug logging has been added to trace it dynamically.

**Confirmed**: The map is EMPTY, not populated with null scores. Snowflake shows perfect correlation between `item_id` presence and `genai_embedding_similarity_score` presence:
- 10-mod stores: `item_id` = populated (e.g. `6289471350`), genai score = present
- 6-mod stores: `item_id` = `""` (empty), genai score = absent

Both `item_id` and `genai_embedding_similarity_score` are read from `generatedRecommendationStoreInfoMap[storeId]` in `updateStoreEntity()`. The fact that BOTH are absent together proves the map lookup returns null (store not in map), not that individual fields are null.

**Per-carousel**: Store 395288 has genai score in `cxgen:2` and `cxgen:4` but NOT in `cxgen:0` — same request, same store, different carousels.

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

## Fix Strategy

The safest fix is to set `genaiEmbeddingSimilarityScore` directly on the entity during `getPostFilterResult()` in `HomepageGeneratedRecommendationProduct`, where `data.storeInfoMap` is directly accessible. This avoids relying on `generatedRecommendationStoreInfoMap` surviving through the SDK pipeline.

### Files to Change

1. **`HomepageGeneratedRecommendationProduct.kt`** — In `getPostFilterResult()`, after filtering/reranking, set `genaiEmbeddingSimilarityScore` on each entity from `data.storeInfoMap`
2. **`HomepageProgrammaticProductServiceUtil.kt`** — Keep `updateStoreEntity()` as-is (belt + suspenders approach)

### Alternative (root cause fix)
Trace exactly where `generatedRecommendationStoreInfoMap` is lost and fix the pipeline. This is harder and riskier since it's deep in the SDK ranking framework.

## Key Files

| File | Purpose |
|---|---|
| `HomepageGeneratedRecommendationProductService.kt` | Fetcher decision (EBR Option 2 DV) |
| `HomepageGeneratedRecommendationProduct.kt:53-134` | `getPostFilterResult()` — reranking + filtering |
| `HomepageProgrammaticCarouselGenerationService.kt:85-171` | Carousel generation + store decoration |
| `HomepageProgrammaticProductServiceUtil.kt:204-228` | `updateStoreEntity()` — sets genai score (FAILS for some stores) |
| `StoreCarouselDataAdapter.kt:2521` | Logging: reads `store.genaiEmbeddingSimilarityScore` |
| `GeneratedRecommendationCarouselService.kt:117-221` | Reranking (alpha^score * beta^similarity) |

## Architecture: Two Fetcher Paths

### Decision point
`HomepageGeneratedRecommendationProductService.fetch()` (line 48):
```kotlin
if (shouldUseGeneratedRecommendationEBROption2) contentSystemsFetcher else fetcher
```

All observed prod events go through the **content systems path** (carousel_details lacks `carousel_rank`, `day_part`).

## Score Flow

```
buildCollection() → generatedRecommendationStoreInfoMap = data.storeInfoMap ✓
  ↓
SDK ranking pipeline (rank → score → sort → postSort) → map preserved via copy() ✓
  ↓
postFilterGeneratedRecommendationCollections → getPostFilterResult → rerank → copy() ✓
  ↓
updateStoreEntity(entity, store, collection)
  → collection.generatedRecommendationStoreInfoMap[productStore.id]?.embeddingScore  ← RETURNS NULL for some stores
```
