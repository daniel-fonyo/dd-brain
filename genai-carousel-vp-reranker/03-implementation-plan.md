# Implementation Plan: Log Embedding Score to cx_cross_vertical_homepage_feed

**Status**: Draft — needs review
**Target table**: `cx_cross_vertical_homepage_feed` (Snowflake, via Iguazu)

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

## Step 5: Populate the Score in the Logging Map

During collection building or store decoration, set the embedding score in the store's logging data map. The score is available in `LiteStoreCollection.generatedRecommendationStoreInfoMap[storeId].embeddingScore`.

**Likely location**: `StoreCarouselDataAdapterUtil.kt` or wherever store-level logging data is assembled for GenAI carousels. Need to trace exactly where the per-store logging map gets built.

## Step 6: Verify Proto Build & Deploy

1. Build proto in services-protobuf, get new version
2. Bump proto dependency in feed-service
3. Deploy to sandbox, verify field appears in Iguazu events
4. Confirm in Snowflake `cx_cross_vertical_homepage_feed` table

## Open Questions

- [ ] Should we also log the **combined reranker score** (`finalScore^alpha * embeddingScore^beta`)? This would be useful for debugging the actual ranking output.
- [ ] Should we log the VP components separately (pConv, pDasherCost, pCommission) as additional fields or use `score_modifiers`?
- [ ] Do we need the same field in `CardView` / `CardClick` protos for client-side logging?
- [ ] Confirm the exact next available field number in `CrossVerticalHomePageFeedEvent`.

## Blockers / Risks

- **Proto deployment lead time**: Proto changes need to be merged and published before feed-service can consume them.
- **Snowflake schema**: Adding a new column to the Snowflake table may require a separate data eng request (or it may auto-populate if the Iguazu pipeline handles new proto fields automatically). Need to confirm.
- **Backfill**: Historical data won't have this field — only new events after deployment will carry it.
