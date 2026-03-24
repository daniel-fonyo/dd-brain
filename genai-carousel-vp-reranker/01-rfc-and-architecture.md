# RFC & Architecture: GenAI Carousel Horizontal Reranker

## RFC Summary

**Authors**: Dipali, Ranjan, Yu Zhang | **Date**: 2026-03-20

### What is EBR?

**Embedding-Based Retrieval**: carousel title + metadata are turned into a text embedding; merchant/item profiles are also embedded; k-NN cosine similarity retrieves the most relevant stores/items per personalized carousel. Computed **online (real-time)**, not offline.

Conceptually: embedding score = **relevance signal**, store ranker score = **conversion signal (pConv)**.

### Current Reranker (Block Reranking)

`reRankStoresByScoreAndSimilarity` in `GeneratedRecommendationCarouselService`:

1. **Read coefficients** from DV per experiment variant: `alpha`, `beta`, `k`
2. **Global sort** by EBR cosine similarity (descending)
3. **Block rerank**: split into chunks of size `k`, within each chunk compute:
   ```
   combinedScore = finalScore^alpha * embeddingScore^beta
   ```
   Sort each chunk by combinedScore descending, stitch back in order.
4. **Return** reordered `LiteStoreCollection`

**Production defaults**: `alpha=0.0`, `beta=1.0`, `k=10` (validated via A/B).
With `alpha=0.0`, `finalScore^0 = 1` — ranker score drops out. Ranking is 100% EBR similarity today.

**Why blocks, not global sort?** Blocks preserve coarse EBR ordering — a low-similarity store can't leapfrog into a higher block. Only stores already deemed relevant compete within a block.

### Proposed Change (VP Value Function)

Replace `finalScore` with an explicit VP score from raw components in `ScoreWithFactors`:

```kotlin
// Current
val combinedScore = finalScore.pow(alpha) * similarity.pow(beta)

// Proposed
val pConv       = scoreWithFactors?.rawScorerScores ?: 0.0
val pDasherCost = scoreWithFactors?.expectedDasherCost ?: 1.0
val pCommission = scoreWithFactors?.expectedCommission ?: 1.0
val vpScore     = pConv * pDasherCost * pCommission
val combinedScore = vpScore.pow(alpha) * similarity.pow(beta)
```

**Formula options under discussion**:

| Option | Formula | Notes |
|--------|---------|-------|
| A | `pConv^alpha * embeddingScore^beta * profitPerConversion` | Keeps embedding score, adds profit multiplier |
| B | Bucket by pConv, rank by finalScore within buckets | No embedding score needed |

### Online A/B Experiment

| Arm | Signal | Description |
|-----|--------|-------------|
| Control | pConv only (`rawScorerScores`) | Clean baseline, no VP multipliers |
| T1 | `pConv * pDasherCost * pCommission` (v1 coefficients) | Full VP formula, set 1 |
| T2 | `pConv * pDasherCost * pCommission` (v2 coefficients) | Full VP formula, set 2 |

All arms share `beta=1.0`, `k=10`. `alpha` set via DV per arm.

---

## Architecture

### Embedding Score Data Flow

```
CRDB / EBR Model
       │
       ▼
CxProfileGeneratedCarousel.storeIdsAndEmbeddingScoreMap
       │
       ▼
GeneratedRecommendationDataService
  ├─ fetchCarouselsWithPreComputedStores()   ← scores from CRDB
  └─ fetchCarouselsWithEbrStores()           ← scores from EBR model call
       │
       ▼
GeneratedRecommendationData.storeInfoMap
  └─ GeneratedRecommendationStoreInfo.embeddingScore   ← per-store score
       │
       ▼
HomepageGeneratedRecommendationProduct.buildCollection()
       │
       ▼
LiteStoreCollection
  ├─ generatedRecommendationStoreInfoMap  ← embedding scores (already here!)
  ├─ storePredictionScoresMap             ← VP/Sibyl scores
  └─ trackingPayload                      ← carousel_rank, day_part (+ alpha/beta after our change)
       │
       ▼
GeneratedRecommendationCarouselService.rerankStoreByBlockReranking()
  reads alpha, beta, k from DV config:
    cx_profile_generated_carousels/reranking_coeff.json
  builds storeSimilarityEmbeddingMap from generatedRecommendationData.storeInfoMap
  combinedScore = finalScore^alpha * embeddingScore^beta
       │
       ▼
HomepageProgrammaticCarouselGenerationService.generateStoreCarousel()
  copies trackingPayload → StoreCarousel.trackingPayload
       │
       ▼
StoreCarouselDataAdapterUtil.generateLogging()  ← container level
  if trackingPayload.isNotEmpty() → writes carousel_details JSON
       │
StoreCarouselDataAdapter.generateStoreLogging()  ← child/store level
  writes per-store fields into logging struct
       │
       ▼
ContainerEventsGenerator → LoggedValue.assign()
  reads keys from logging struct → calls proto setters
       │
       ▼
cx_cross_vertical_homepage_feed (Snowflake)
```

### Two Retrieval Paths

| Path | Source | trackingPayload? | Controlled By |
|------|--------|-----------------|---------------|
| A: Direct fetcher | CRDB / in-process | Works | Experiment flag |
| B: Content Systems RPC (EBR) | EBR model via RPC | **DROPPED** — serializer omits it | Default for most traffic |

**Path B bug**: `GeneratedRecommendationCarouselsSerializer.serialize()` does not serialize `trackingPayload`. The deserializer in `GeneratedRecommendationContentSystemsFetcherUtil.toGeneratedRecommendationData()` defaults it to `emptyMap()`. This is why `carousel_details = {}` in Snowflake for most GenAI traffic.

### Key Files — Ranking

| File (feed-service) | Purpose |
|------|---------|
| `libraries/discovery/.../GeneratedRecommendationCarouselService.kt` | `rerankStoreByBlockReranking()` — block reranking (line ~138-210). Alpha/beta/k read at line ~148-153 via `DiscoveryRuntimeUtil.getRerankingCoefficientsForCxProfile()` |
| `libraries/discovery/.../HomepageGeneratedRecommendationProduct.kt` | Invokes reranking (line ~103), builds collection with `.trackingPayload(data.trackingPayload)` (line ~196) |
| `libraries/domain-util/.../contentsystems/GeneratedRecommendationDataService.kt` | EBR retrieval, carousel creation, sets initial `trackingPayload` (line ~551-555) |
| `libraries/discovery-utils/.../models/GeneratedRecommendationData.kt` | `GeneratedRecommendationStoreInfo(storeId, itemId, embeddingScore, imageUrl)` |
| `libraries/domain-util/.../carousel/models/collections/LiteStoreCollection.kt` | `generatedRecommendationStoreInfoMap` (line 92), `trackingPayload` (line 94), `storePredictionScoresMap` (line 58) |

### Key Files — Logging

| File | Purpose |
|------|---------|
| `services-protobuf: protos/feed_service/events.proto` | `CrossVerticalHomePageFeedEvent` (line 80-219) — proto schema |
| `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt` | `LoggedValue` enum: key → proto setter. Child logging (~line 266-311), container logging (~line 160-209) |
| `libraries/domain-util/.../facets/adapters/StoreCarouselDataAdapter.kt` | `generateStoreLogging()` (line ~2337) — per-store logging struct. Writes `expected_commission`/`expected_dasher_cost` at line ~2510-2516 |
| `libraries/domain-util/.../carousel/utils/StoreCarouselDataAdapterUtil.kt` | `generateLogging()` — container level. Writes `carousel_details` from `trackingPayload` (line ~663-672) |
| `libraries/domain-util/.../utils/logging/DomainUtilLoggingConstants.kt` | Key string constants |
| `libraries/discovery/.../programmatic/util/HomepageProgrammaticProductServiceUtil.kt` | `updateStoreEntity()` (line ~204) — copies ScoreWithFactors fields to StoreEntity |
| `pipelines/homepage/.../HomepageProgrammaticCarouselGenerationService.kt` | `generateStoreCarousel()` (line ~561) — copies `trackingPayload` to StoreCarousel (line ~599) |

### Existing score_modifiers Pattern (expected_commission trace)

```
ScoreWithFactors.expectedCommission
  → StoreCarouselService.decorateEntities() copies to StoreEntity.expectedCommission
  → StoreCarouselDataAdapter.generateStoreLogging() writes to struct[EXPECTED_COMMISSION_KEY]
  → LoggedValue.EXPECTED_VP_SCORE reads struct, calls builder.addScoreModifiers(name, value)
  → Appears in score_modifiers JSON array in Snowflake
```

### What's Already Logged (CrossVerticalHomePageFeedEvent)

| Proto Field | # | What It Logs |
|-------------|---|--------------|
| `horizontal_element_score` | 19 | VP prediction score per store |
| `raw_horizontal_element_score` | 80 | Raw Sibyl score pre-multipliers |
| `facet_score` | 20 | Carousel-level aggregate score |
| `raw_facet_score` | 79 | Raw carousel score |
| `score_modifiers` | 59 | Named score components (VP, dasher cost, uncertainty, etc.) |
| `carousel_details` | 81 | JSON: `{carousel_id, carousel_rank, day_part, last_update_date}` |
| `predictor_names` / `model_ids` | 57/58 | Model predictor names and IDs |

### What's NOT Logged (The Gap)

| Signal | Level | Target |
|--------|-------|--------|
| **EBR embedding similarity score** | Per store | → `score_modifiers` |
| **Alpha** | Per carousel | → `carousel_details` via `trackingPayload` |
| **Beta** | Per carousel | → `carousel_details` via `trackingPayload` |
| **Future formula params** | Per carousel | → `carousel_details` via `trackingPayload` (auto) |
