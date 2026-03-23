# Architecture: Embedding Scores & Logging in feed-service

## Embedding Score Data Flow

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
  combinedScore = finalScore^alpha * embeddingScore^beta
       │
       ▼
Logging pipeline (IguazuEventUtil → ContainerEventsGenerator)
       │
       ▼
cx_cross_vertical_homepage_feed (Snowflake)
```

## Two Retrieval Paths

| Path | Source | Controlled By |
|------|--------|---------------|
| Pre-computed | CRDB `storeIdsAndEmbeddingScoreMap` | Experiment flag |
| Real-time EBR | EBR model service (k-NN on embeddings) | Default path |

## Key Files — Ranking

| File | Key Code |
|------|----------|
| `libraries/discovery/src/main/kotlin/.../GeneratedRecommendationCarouselService.kt` | `rerankStoreByBlockReranking()` — the block reranking algorithm (line ~138-210) |
| `libraries/discovery/src/main/kotlin/.../HomepageGeneratedRecommendationProduct.kt` | Invokes reranking (line ~103), builds collection (line ~195) |
| `libraries/domain-util/src/main/kotlin/.../contentsystems/GeneratedRecommendationDataService.kt` | EBR retrieval, carousel creation, embedding extraction |
| `libraries/domain-util/src/main/kotlin/.../contentsystems/GeneratedRecommendationWorkflowService.kt` | Orchestrates pre-computed vs EBR paths |
| `libraries/discovery-utils/src/main/kotlin/.../models/GeneratedRecommendationData.kt` | `GeneratedRecommendationStoreInfo(storeId, itemId, embeddingScore, imageUrl)` |
| `libraries/domain-util/src/main/kotlin/.../carousel/models/collections/LiteStoreCollection.kt` | Holds `generatedRecommendationStoreInfoMap` and `storePredictionScoresMap` |

## Key Files — Logging

| File | Key Code |
|------|----------|
| `services-protobuf/protos/feed_service/events.proto` | `CrossVerticalHomePageFeedEvent` (line 80-219) — the proto schema |
| `libraries/domain-util/src/main/kotlin/.../iguazu/IguazuEventUtil.kt` | Event generation entry point, event name = `cx_cross_vertical_homepage_feed` |
| `libraries/domain-util/src/main/kotlin/.../iguazu/ContainerEventsGenerator.kt` | `LoggedValue` enum maps keys → proto setters. Child logging (line ~266-311), container logging (line ~160-209) |
| `libraries/domain-util/src/main/kotlin/.../carousel/utils/StoreCarouselDataAdapterUtil.kt` | Populates `carousel_details` JSON from `trackingPayload` (line ~663-671) |
| `libraries/domain-util/src/main/kotlin/.../utils/logging/DomainUtilLoggingConstants.kt` | Logging key constants |

## Existing Logged Scores (CrossVerticalHomePageFeedEvent)

| Proto Field | Field # | What It Logs | Source |
|-------------|---------|--------------|--------|
| `horizontal_element_score` | 19 | VP prediction score per store | `STORE_PREDICTION_SCORE` → `setHorizontalElementScore()` |
| `raw_horizontal_element_score` | 80 | Raw Sibyl score (pre-multipliers) | `STORE_RAW_PREDICTION_SCORE` → `setRawHorizontalElementScore()` |
| `facet_score` | 20 | Carousel-level aggregate score | `CONTAINER_SCORE` → `setFacetScore()` |
| `raw_facet_score` | 79 | Raw carousel score | `CONTAINER_RAW_SCORE` → `setRawFacetScore()` |
| `score_modifiers` | 59 | Named score components (VP, dasher cost, uncertainty) | `ScoreModifier(name, value)` |
| `carousel_details` | 81 | JSON: `{carousel_id, carousel_rank, day_part, last_update_date}` | `trackingPayload` |
| `predictor_names` | 57 | Model predictor names | `STORE_PREDICTOR_NAMES` |
| `model_ids` | 58 | Model IDs used | `STORE_PREDICTOR_MODEL_IDS` |

## What's Missing (The Gap)

| Signal | Status | Notes |
|--------|--------|-------|
| EBR embedding similarity score | **NOT LOGGED** | Per-store cosine similarity; used as `embeddingScore` in reranker |
| Combined reranker score | **NOT LOGGED** | `finalScore^alpha * embeddingScore^beta` — the actual horizontal ranking signal |
| VP components (pConv, pDasherCost, pCommission) | Partially logged | `score_modifiers` has some, but not cleanly separated for GenAI context |

## How VP Scores Were Added (Prior Art)

The pattern for adding a new logged score:

1. **Proto**: Add field to `CrossVerticalHomePageFeedEvent` in `events.proto`
2. **Constant**: Add key string in `DomainUtilLoggingConstants.kt`
3. **LoggedValue enum**: Add entry in `ContainerEventsGenerator.kt` mapping key → proto setter
4. **Include in child logging list**: Add to the `childLoggedValues` list in `ContainerEventsGenerator`
5. **Populate the key**: Set the value in the store's logging map during collection building or decoration

Example: `STORE_PREDICTION_SCORE` maps `STORE_PREDICTION_SCORE_KEY` → `setHorizontalElementScore()` and is included in child logging at line ~266-311.
