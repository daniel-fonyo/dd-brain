# Implementation Plan: GenAI Reranker Signal Logging

**Status**: Draft — needs review
**Target table**: `cx_cross_vertical_homepage_feed` (Snowflake, via Iguazu)
**Urgency**: High — GenAI V1 live on 60% of orders; need 1+ week of data before experiments.

## What to Log

| Signal | Granularity | Priority |
|--------|-------------|----------|
| EBR embedding similarity score | Per store × carousel × consumer | **Must have** |
| Alpha (ranker score exponent, from DV) | Per carousel | **Must have** |
| Beta (embedding score exponent, from DV) | Per carousel | **Must have** |
| Any future formula param (gamma, etc.) | Per carousel | **Auto-logged by design** |

## Approach: Extensible Reranker Params Map

The formula will evolve (VP multipliers, gamma, new formulas). Dipali's requirement: "Whatever is used to rank should always be logged, even if the formula changes."

**Recommended — Hybrid**:
- **Dedicated proto field** for `embedding_similarity_score` (per-store, high-value, queried directly in SQL)
- **Extensible JSON map** for reranker formula params (alpha, beta, and any future coefficients). Use the existing `carousel_details` JSON pattern or a new `reranker_params` JSON field. This way, adding a new coefficient to the reranker only requires adding it to the map at the ranking call site — no proto/logging plumbing changes needed.

## Steps

### 1. Proto Change (services-protobuf)

Add to `CrossVerticalHomePageFeedEvent` in `events.proto`:

```protobuf
double embedding_similarity_score = <next_field_number>;
```

**File**: `protos/feed_service/events.proto`
**Action**: Check next available field number (currently up to ~85+).

### 2. Logging Constant (feed-service)

Add in `DomainUtilLoggingConstants.kt`:

```kotlin
const val EMBEDDING_SIMILARITY_SCORE_KEY = "embedding_similarity_score"
```

### 3. LoggedValue Enum Entry (feed-service)

Add in `ContainerEventsGenerator.kt`:

```kotlin
EMBEDDING_SIMILARITY_SCORE(
    EMBEDDING_SIMILARITY_SCORE_KEY,
    { builder, value -> builder.setEmbeddingSimilarityScore(value.toDouble()) }
)
```

### 4. Include in Child Logging List

Add `LoggedValue.EMBEDDING_SIMILARITY_SCORE` to the per-store logged values list in `ContainerEventsGenerator` (~line 266-311).

### 5. Populate Embedding Score in Logging Map

Set the score from `LiteStoreCollection.generatedRecommendationStoreInfoMap[storeId].embeddingScore` into the store's logging data map.

**Likely location**: `StoreCarouselDataAdapterUtil.kt` or wherever store-level logging data is assembled for GenAI carousels. Need to trace where the per-store logging map is built.

### 6. Add Reranker Params to trackingPayload

In `rerankStoreByBlockReranking()` (or where DV values are read), extend `trackingPayload` with all formula params. These flow through to `carousel_details` JSON automatically.

```kotlin
val trackingPayload = mapOf(
    "carousel_rank" to carousel.rank,
    "day_part" to retrievalContext.dayPart,
    "last_update_date" to (carousel.lastUpdateDate ?: ""),
    "reranker_alpha" to alpha,
    "reranker_beta" to beta,
    // future: "reranker_gamma" to gamma — just add here, auto-logged
)
```

This is the extensibility point — any new formula param just needs to be added to this map, no proto or logging plumbing changes.

### 7. Verify

1. Build proto in services-protobuf, publish new version
2. Bump proto dependency in feed-service
3. Deploy to sandbox, verify field appears in Iguazu events
4. Confirm in Snowflake `cx_cross_vertical_homepage_feed` table
5. Verify `carousel_details` JSON contains `reranker_alpha`, `reranker_beta`

## Open Questions

- [ ] Confirm `horizontal_element_score` populates for GenAI programmatic carousels.
- [ ] Do we need `embedding_similarity_score` in `CardView` / `CardClick` protos for client-side logging?
- [ ] Confirm next available field number in `CrossVerticalHomePageFeedEvent`.
- [ ] Does Snowflake auto-pick up new proto fields or do we need a data eng request?

## Blockers / Risks

- **Proto deployment lead time**: Proto changes need to merge and publish before feed-service can consume them.
- **Snowflake schema**: New proto field may need a data eng request to surface as a column, or it may auto-populate. Need to confirm.
- **Backfill**: Historical data won't have this field — only new events after deployment.
