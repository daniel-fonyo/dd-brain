# Implementation Plan: Log Embedding Score to cx_cross_vertical_homepage_feed

**Status**: Draft — needs review
**Target table**: `cx_cross_vertical_homepage_feed` (Snowflake, via Iguazu)
**Urgency**: High — GenAI V1 is live on 60% of orders; Dipali needs 1+ week of data before experiments can start.

## What to Log

Per meeting agreement (2026-03-20):

| Signal | Granularity | Priority |
|--------|-------------|----------|
| Embedding similarity score | Per store, per carousel, per consumer | **Must have** |
| Alpha (ranker score exponent) | Per carousel (from DV) | **Must have** |
| Beta (embedding score exponent) | Per carousel (from DV) | **Must have** |
| Future formula params (gamma, etc.) | TBD | **Design for extensibility** |

The store ranker score (`finalScore`) is already logged as `horizontal_element_score` — confirm it populates for GenAI programmatic carousels.

## Approach: Extensible JSON Field vs Dedicated Proto Fields

Dipali wants a scalable approach — she'll change the formula over time (add VP, gamma, etc.) and doesn't want to re-plumb each time.

**Option A — Dedicated proto fields**: One field per signal. Clean SQL queries but requires proto change for every new param.

**Option B — JSON in `carousel_details`**: Extend the existing `carousel_details` JSON (already has `carousel_rank`, `day_part`) with ranking params. Extensible but harder to query.

**Option C (Recommended) — Hybrid**: Dedicated proto field for `embedding_similarity_score` (high-value, per-store, queried frequently) + extend `carousel_details` JSON with `alpha`, `beta`, and future formula params (per-carousel, less frequently queried individually). This balances queryability with extensibility.

## Step 1: Proto Change (services-protobuf)

Add a new field to `CrossVerticalHomePageFeedEvent` in `events.proto`:

```protobuf
// EBR embedding similarity score used for GenAI carousel horizontal ranking
double embedding_similarity_score = <next_field_number>;
```

**File**: `protos/feed_service/events.proto`
**Branch**: `feat/embedding-score-logging` in services-protobuf
**Note**: Check the next available field number (currently fields go up to ~85+).

## Step 2: Logging Constant (feed-service)

Add a new constant in `DomainUtilLoggingConstants.kt`:

```kotlin
const val EMBEDDING_SIMILARITY_SCORE_KEY = "embedding_similarity_score"
```

**File**: `libraries/domain-util/.../utils/logging/DomainUtilLoggingConstants.kt`

## Step 3: LoggedValue Enum Entry (feed-service)

Add a new `LoggedValue` entry in `ContainerEventsGenerator.kt`:

```kotlin
EMBEDDING_SIMILARITY_SCORE(
    EMBEDDING_SIMILARITY_SCORE_KEY,
    { builder, value -> builder.setEmbeddingSimilarityScore(value.toDouble()) }
)
```

**File**: `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt`

## Step 4: Include in Child Logging List

Add `LoggedValue.EMBEDDING_SIMILARITY_SCORE` to the child (per-store) logged values list in `ContainerEventsGenerator`.

**Location**: `ContainerEventsGenerator.kt`, child logging list (~line 266-311)

## Step 5: Add Alpha/Beta to carousel_details (or trackingPayload)

Extend the `trackingPayload` in `GeneratedRecommendationDataService.createGeneratedRecommendationCarousels()` to include `alpha` and `beta` values read from DV. These will flow through to `carousel_details` JSON automatically.

```kotlin
val trackingPayload = mapOf(
    "carousel_rank" to carousel.rank,
    "day_part" to retrievalContext.dayPart,
    "last_update_date" to (carousel.lastUpdateDate ?: ""),
    "reranker_alpha" to alpha,
    "reranker_beta" to beta,
)
```

**Location**: `GeneratedRecommendationDataService.kt` or `GeneratedRecommendationCarouselService.kt` (where DV values are read).

## Step 6: Populate the Embedding Score in the Logging Map

During collection building or store decoration, set the embedding score in the store's logging data map. The score is available in `LiteStoreCollection.generatedRecommendationStoreInfoMap[storeId].embeddingScore`.

**Likely location**: `StoreCarouselDataAdapterUtil.kt` or wherever store-level logging data is assembled for GenAI carousels. Need to trace exactly where the per-store logging map gets built.

## Step 7: Verify Proto Build & Deploy

1. Build proto in services-protobuf, get new version
2. Bump proto dependency in feed-service
3. Deploy to sandbox, verify field appears in Iguazu events
4. Confirm in Snowflake `cx_cross_vertical_homepage_feed` table

## Open Questions

- [x] ~~Should we log alpha/beta?~~ Yes — agreed in meeting.
- [ ] Confirm `horizontal_element_score` populates for GenAI programmatic carousels (Dipali wasn't sure).
- [ ] Do we need the same field in `CardView` / `CardClick` protos for client-side logging?
- [ ] Confirm the exact next available field number in `CrossVerticalHomePageFeedEvent`.
- [ ] Does Snowflake auto-pick up new proto fields or do we need a data eng request?

## Blockers / Risks

- **Proto deployment lead time**: Proto changes need to be merged and published before feed-service can consume them.
- **Snowflake schema**: Adding a new column to the Snowflake table may require a separate data eng request (or it may auto-populate if the Iguazu pipeline handles new proto fields automatically). Need to confirm.
- **Backfill**: Historical data won't have this field — only new events after deployment will carry it.
