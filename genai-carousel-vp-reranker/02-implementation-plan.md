# Implementation Plan: GenAI Reranker Signal Logging

**Status**: Ready to implement
**Target table**: `cx_cross_vertical_homepage_feed` (Snowflake, via Iguazu)
**Urgency**: High — GenAI V1 live on 60% of orders; need 1+ week of data before experiments.
**Proto changes to events.proto**: None required.

## Approach

Two existing logging channels, matched to data grain:

| Signal | Grain | Channel | Extensibility |
|--------|-------|---------|--------------|
| Embedding score | Per store | `score_modifiers` (LoggedValue pattern) | New per-store signals = new LoggedValue entry |
| Alpha, beta, future params | Per carousel | `carousel_details` JSON (via `trackingPayload`) | New param = one line in reranker map. Auto-logged. |

## Changes by File

### Part A: Embedding Score → score_modifiers

**A1. `StoreEntity.kt`** — add field (~1 line)
```kotlin
val embeddingScore: Double? = null,
```

**A2. `HomepageProgrammaticProductServiceUtil.kt`** — copy in `updateStoreEntity()` (~1 line)
```kotlin
embeddingScore = collection.generatedRecommendationStoreInfoMap[productStore.id]?.embeddingScore,
```

**A3. `StoreCarouselService.kt`** — copy in `decorateEntities()` (~1 line)
```kotlin
embeddingScore = liteStoreCollection.generatedRecommendationStoreInfoMap[it.id]?.embeddingScore,
```

**A4. `StoreCarouselDataAdapter.kt`** — write to logging struct in `generateStoreLogging()` (~3 lines)
```kotlin
store.embeddingScore?.let {
    map[EMBEDDING_SIMILARITY_SCORE_KEY] = getSafeNumberValueWithDefault(it)
}
```

**A5. `DomainUtilLoggingConstants.kt`** — add key constant (~1 line)
```kotlin
const val EMBEDDING_SIMILARITY_SCORE_KEY = "embedding_similarity_score"
```

**A6. `ContainerEventsGenerator.kt`** — add LoggedValue enum entry + include in child list (~20 lines)
```kotlin
EMBEDDING_SIMILARITY_SCORE(EMBEDDING_SIMILARITY_SCORE_KEY, { b, v ->
    when (b) {
        is Events.CrossVerticalHomePageFeedEvent.Builder ->
            b.addScoreModifiers(
                Events.CrossVerticalHomePageFeedEvent.ScoreModifier.newBuilder()
                    .setName(EMBEDDING_SIMILARITY_SCORE_KEY).setValue(v.numberValue).build(),
            )
        else -> b
    }
}),
```
Then add `LoggedValue.EMBEDDING_SIMILARITY_SCORE` to child logging lists.

### Part B: Alpha/Beta → carousel_details via trackingPayload

**B1. `GeneratedRecommendationCarouselService.kt`** — in `rerankStoreByBlockReranking()`, merge alpha/beta into trackingPayload on the returned collection (~5 lines)
```kotlin
return collection.copy(
    entities = combinedStores,
    storeIds = combinedStores.map { it.storeId() },
    trackingPayload = collection.trackingPayload + mapOf(
        "reranker_alpha" to alpha,
        "reranker_beta" to beta,
        // future gamma etc. — just add here, auto-logged via carousel_details
    ),
)
```

No other changes needed — `trackingPayload` already flows through:
`LiteStoreCollection` → `generateStoreCarousel()` → `StoreCarousel.trackingPayload` → `StoreCarouselDataAdapterUtil.generateLogging()` → `carousel_details` JSON.

### Part C: Fix Path B trackingPayload Drop (Content Systems RPC)

`carousel_details` is `{}` for GenAI traffic on the Content Systems / EBR path because the serializer/deserializer drops `trackingPayload`.

**C1. `content_systems.proto`** (services-protobuf) — add field to `GeneratedRecommendationCarousel`
```protobuf
map<string, string> tracking_payload = <next_field>;
```

**C2. `GeneratedRecommendationCarouselsSerializer.kt`** — serialize trackingPayload into proto

**C3. `GeneratedRecommendationContentSystemsFetcherUtil.kt`** — deserialize trackingPayload from proto
```kotlin
trackingPayload = carousel.trackingPayloadMap,
```

**Note**: Part C requires a proto change in `content_systems.proto` (internal RPC proto, not logging proto — smaller blast radius). Could be deferred if Path A has enough traffic, but needed for full coverage.

## Summary

| File | Part | Lines |
|------|------|-------|
| `StoreEntity.kt` | A1 | ~1 |
| `HomepageProgrammaticProductServiceUtil.kt` | A2 | ~1 |
| `StoreCarouselService.kt` | A3 | ~1 |
| `StoreCarouselDataAdapter.kt` | A4 | ~3 |
| `DomainUtilLoggingConstants.kt` | A5 | ~1 |
| `ContainerEventsGenerator.kt` | A6 | ~20 |
| `GeneratedRecommendationCarouselService.kt` | B1 | ~5 |
| `content_systems.proto` | C1 | ~1 |
| `GeneratedRecommendationCarouselsSerializer.kt` | C2 | ~3 |
| `GeneratedRecommendationContentSystemsFetcherUtil.kt` | C3 | ~2 |

**Parts A + B**: ~32 lines across 7 files in feed-service. Zero proto changes.
**Part C**: ~6 lines across 3 files (1 proto + 2 Kotlin). Needed for full coverage.

## Verification

1. Deploy to sandbox
2. Hit homepage, check Iguazu events contain:
   - `score_modifiers` array includes `{"name": "embedding_similarity_score", "value": ...}`
   - `carousel_details` JSON includes `reranker_alpha` and `reranker_beta`
3. Confirm in Snowflake `cx_cross_vertical_homepage_feed`
4. Share sample query with Dipali for validation

## Open Questions

- [x] ~~Where to log embedding score?~~ → `score_modifiers` (per-store, no proto change)
- [x] ~~Where to log alpha/beta?~~ → `carousel_details` via `trackingPayload` (per-carousel, auto-extensible)
- [x] ~~Does `horizontal_element_score` populate for GenAI?~~ → Yes, confirmed from Snowflake data
- [ ] Can Part C be deferred? Depends on Path A vs Path B traffic split.
- [ ] Do we need embedding score in `CardView`/`CardClick` protos? (Likely no for initial ask)
