# Implementation Plan: GenAI Reranker Signal Logging

**Status**: Implemented — pending sandbox test
**Branch**: `feed-service: feat/genai-reranker-logging`
**Target table**: `cx_cross_vertical_homepage_feed` (Snowflake, via Iguazu)
**Urgency**: High — GenAI V1 live on 60% of orders; need 1+ week of data before experiments.
**Proto changes**: None required.

## Approach

Two existing logging channels, matched to data grain:

| Signal | Grain | Channel | Extensibility |
|--------|-------|---------|--------------|
| Embedding score | Per store | `score_modifiers` (LoggedValue pattern) | New per-store signals = new LoggedValue entry |
| Alpha, beta, future params | Per carousel | `carousel_details` JSON (via `trackingPayload`) | New param = one line in reranker map. Auto-logged. |

Both channels work on **100% of traffic** (Path A + Path B):
- `embeddingScore` is in `content_systems.proto` — both paths deliver it to `generatedRecommendationStoreInfoMap`
- Alpha/beta are read from DV inside feed-service's reranker — independent of fetch path
- `trackingPayload` is set in the reranker (our change), which runs after both paths converge

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
    map[GENAI_EMBEDDING_SIMILARITY_SCORE_KEY] = getSafeNumberValueWithDefault(it)
}
```

**A5. `DomainUtilLoggingConstants.kt`** — add key constant (~1 line)
```kotlin
const val GENAI_EMBEDDING_SIMILARITY_SCORE_KEY = "genai_embedding_similarity_score"
```

**A6. `ContainerEventsGenerator.kt`** — add LoggedValue enum entry + include in child list (~20 lines)
```kotlin
GENAI_EMBEDDING_SIMILARITY_SCORE(GENAI_EMBEDDING_SIMILARITY_SCORE_KEY, { b, v ->
    when (b) {
        is Events.CrossVerticalHomePageFeedEvent.Builder ->
            b.addScoreModifiers(
                Events.CrossVerticalHomePageFeedEvent.ScoreModifier.newBuilder()
                    .setName(GENAI_EMBEDDING_SIMILARITY_SCORE_KEY).setValue(v.numberValue).build(),
            )
        else -> b
    }
}),
```
Then add `LoggedValue.GENAI_EMBEDDING_SIMILARITY_SCORE` to child logging lists.

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
```
LiteStoreCollection.trackingPayload
  → generateStoreCarousel() copies to StoreCarousel.trackingPayload
  → StoreCarouselDataAdapterUtil.generateLogging() writes to carousel_details JSON
  → ContainerEventsGenerator reads CAROUSEL_DETAILS from container logging struct
  → Appears in carousel_details column in Snowflake
```

**Note**: On Path B traffic, the original `trackingPayload` from Content Systems is empty (pre-existing gap — `carousel_rank`, `day_part` missing). But our alpha/beta values are injected by the reranker *after* fetch, so they will appear regardless. The carousel_details JSON will contain `{carousel_id, reranker_alpha, reranker_beta}` on Path B (without carousel_rank/day_part), and the full set on Path A.

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

**Total: ~32 lines across 7 files. Zero proto changes.**

## Sandbox DV Overrides

To see GenAI carousels with the reranker in sandbox:

| DV | Type | Value | Notes |
|----|------|-------|-------|
| `should_enable_discovery_flow_in_homepage` | Boolean | `true` | Gates entire discovery SDK flow |
| `enable_generated_recommendation_product` | String | `ebr_cx_profile_store_first` | Enables GenAI carousels. Any `ebr_*` arm works |
| `cx_homepage_programmatic_carousel_dropping` | String | `control` | Must NOT be `treatment_2` |
| `enable_generated_recommendation_product_mock` | String | `treatment` | Only if CRDB has no data for test consumer |

Runtime config (already synced to sandbox):
| Path | Value |
|------|-------|
| `cx_profile_generated_carousels/reranking_coeff.json` | Must have entry for variant, e.g. `{"ebr_cx_profile_store_first": [0.0, 1.0, 10]}` |

Test consumer `757606047L` (per CLAUDE.md) must have data in `cx_profile_generated_carousels` CRDB tables, or use mock data DV.

## Verification

1. Deploy to sandbox
2. Hit homepage, check Iguazu events contain:
   - `score_modifiers` array includes `{"name": "genai_embedding_similarity_score", "value": ...}`
   - `carousel_details` JSON includes `reranker_alpha` and `reranker_beta`
3. Confirm in Snowflake `cx_cross_vertical_homepage_feed`
4. Share sample query with Dipali for validation

## Open Questions

- [x] ~~Where to log embedding score?~~ → `score_modifiers`
- [x] ~~Where to log alpha/beta?~~ → `carousel_details` via `trackingPayload`
- [x] ~~Does `horizontal_element_score` populate for GenAI?~~ → Yes
- [x] ~~Does embedding score flow through Path B?~~ → Yes, it's in `content_systems.proto`
- [x] ~~Do alpha/beta work on Path B?~~ → Yes, read from DV in reranker after fetch converges
- [ ] Do we need embedding score in `CardView`/`CardClick` protos? (Likely no for initial ask)
