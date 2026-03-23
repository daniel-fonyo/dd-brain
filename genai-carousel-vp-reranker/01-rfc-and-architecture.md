# RFC & Architecture: GenAI Carousel Horizontal Reranker

## RFC Summary

**Authors**: Dipali, Ranjan, Yu Zhang | **Date**: 2026-03-20

### What is EBR?

**Embedding-Based Retrieval**: carousel title + metadata are turned into a text embedding; merchant/item profiles are also embedded; k-NN cosine similarity retrieves the most relevant stores/items per personalized carousel. Dipali believes this is computed **online (real-time)**, not offline.

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

Also considering: calibrate SR score first; use rank (not score) for chunk-wise ranking.

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
  ├─ generatedRecommendationStoreInfoMap  ← embedding scores
  ├─ storePredictionScoresMap             ← VP/Sibyl scores
  └─ trackingPayload                      ← carousel_rank, day_part
       │
       ▼
GeneratedRecommendationCarouselService.rerankStoreByBlockReranking()
  reads alpha, beta, k from DV
  combinedScore = finalScore^alpha * embeddingScore^beta
       │
       ▼
Logging pipeline (IguazuEventUtil → ContainerEventsGenerator)
       │
       ▼
cx_cross_vertical_homepage_feed (Snowflake)
```

### Two Retrieval Paths

| Path | Source | Controlled By |
|------|--------|---------------|
| Pre-computed | CRDB `storeIdsAndEmbeddingScoreMap` | Experiment flag |
| Real-time EBR | EBR model service (k-NN on embeddings) | Default path |

### Key Files — Ranking

| File (feed-service) | Purpose |
|------|---------|
| `libraries/discovery/.../GeneratedRecommendationCarouselService.kt` | `rerankStoreByBlockReranking()` — block reranking algorithm (line ~138-210) |
| `libraries/discovery/.../HomepageGeneratedRecommendationProduct.kt` | Invokes reranking (line ~103), builds collection (line ~195) |
| `libraries/domain-util/.../contentsystems/GeneratedRecommendationDataService.kt` | EBR retrieval, carousel creation, embedding extraction |
| `libraries/domain-util/.../contentsystems/GeneratedRecommendationWorkflowService.kt` | Orchestrates pre-computed vs EBR paths |
| `libraries/discovery-utils/.../models/GeneratedRecommendationData.kt` | `GeneratedRecommendationStoreInfo(storeId, itemId, embeddingScore, imageUrl)` |
| `libraries/domain-util/.../carousel/models/collections/LiteStoreCollection.kt` | Holds `generatedRecommendationStoreInfoMap` and `storePredictionScoresMap` |

### Key Files — Logging

| File | Purpose |
|------|---------|
| `services-protobuf: protos/feed_service/events.proto` | `CrossVerticalHomePageFeedEvent` (line 80-219) — proto schema |
| `libraries/domain-util/.../iguazu/IguazuEventUtil.kt` | Event generation, event name = `cx_cross_vertical_homepage_feed` |
| `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt` | `LoggedValue` enum: key → proto setter. Child logging (~line 266-311), container logging (~line 160-209) |
| `libraries/domain-util/.../carousel/utils/StoreCarouselDataAdapterUtil.kt` | Populates `carousel_details` JSON from `trackingPayload` (line ~663-671) |
| `libraries/domain-util/.../utils/logging/DomainUtilLoggingConstants.kt` | Logging key constants |

### What's Already Logged (CrossVerticalHomePageFeedEvent)

| Proto Field | # | What It Logs |
|-------------|---|--------------|
| `horizontal_element_score` | 19 | VP prediction score per store (via `setHorizontalElementScore()`) |
| `raw_horizontal_element_score` | 80 | Raw Sibyl score pre-multipliers |
| `facet_score` | 20 | Carousel-level aggregate score |
| `raw_facet_score` | 79 | Raw carousel score |
| `score_modifiers` | 59 | Named score components (VP, dasher cost, uncertainty) |
| `carousel_details` | 81 | JSON: `{carousel_id, carousel_rank, day_part, last_update_date}` |
| `predictor_names` / `model_ids` | 57/58 | Model predictor names and IDs |

### What's NOT Logged (The Gap)

| Signal | Notes |
|--------|-------|
| **EBR embedding similarity score** | Per-store cosine similarity; the primary horizontal ranking signal today |
| **Alpha** | DV-controlled exponent for ranker score in reranker formula |
| **Beta** | DV-controlled exponent for embedding score in reranker formula |
| **Future formula params** | Any new coefficient (gamma, etc.) added to the reranker formula |

The store ranker score (`finalScore`) is logged as `horizontal_element_score`, but the reranker's own inputs — the signals it actually uses to determine horizontal order — are invisible.

### Prior Art: How VP Scores Were Added

Pattern for adding a new logged score:

1. **Proto**: Add field to `CrossVerticalHomePageFeedEvent` in `events.proto`
2. **Constant**: Add key string in `DomainUtilLoggingConstants.kt`
3. **LoggedValue enum**: Add entry in `ContainerEventsGenerator.kt` mapping key → proto setter
4. **Child logging list**: Add to `childLoggedValues` in `ContainerEventsGenerator`
5. **Populate**: Set value in store's logging map during collection building or decoration

Example: `STORE_PREDICTION_SCORE` maps `STORE_PREDICTION_SCORE_KEY` → `setHorizontalElementScore()`, included in child logging at line ~266-311.
