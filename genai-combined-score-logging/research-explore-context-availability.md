# Research: ExploreContext Availability Across Score Computation Sites

**Date**: 2026-03-27
**Purpose**: Verify that `ExploreContext` (via `BaseDiscoveryProductContext`) is accessible everywhere ranking signals are currently computed or logged, to validate the RankingSignalCollector design.

## ExploreContext Overview

- **Class**: `ExploreContext` in `libraries/sdk-dex/.../models/context/ExploreContext.kt`
- **Type**: Kotlin data class, 43 fields, implements `BaseDiscoveryProductContext`
- **Created by**: `HomepageRequestToContext.transform(request)` at pipeline entry
- **Threading**: Passed as an explicit parameter through suspend function chains
- **Mutable state**: `debuggingMap: MutableMap<String, Any>` is the only mutable field. The proposed `rankingSignalCollector` follows the same pattern (mutable object held by immutable reference).

## Pipeline Flow Verification

### Pipeline Entry
```
DefaultHomePagePipeline.constructWorkflow(request)
  -> defaultHomepageRequestToContext.transform(request) -> creates ExploreContext
  -> passed as "homepageFeedContext" to all downstream jobs
```

### Post Processor
```
DefaultHomePagePostProcessor.reOrder(context: ExploreContext, layout)
  -> rankAndDedupeContent(context: ExploreContext, collections, storeCarousels, dealCarousel, ...)
    -> rankAndMergeContent(context: ExploreContext, content: RankableContent)
```

### GenAI Reranking
```
GeneratedRecommendationCarouselService.reRankStoresByScoreAndSimilarity(
    collection: LiteStoreCollection,
    generatedRecommendationData: GeneratedRecommendationData,
    context: BaseDiscoveryProductContext  <- ExploreContext passed here
)
```

### Store Adapter
```
StoreCarouselDataAdapter.toFacetV2(storeCarousel, context: ExploreContext, ...)
  -> generateStoreLogging(carouselId: String, context: ExploreContext, store: StoreEntity, ...)
```

### Deal Adapter
```
DealCarouselDataAdapter.toFacetV2(dealCarousel, context: ExploreContext)
```

### Item Adapter
```
ItemCarouselDataAdapter.toFacetV2(itemCarousel, context: ExploreContext, ...)
```

### Iguazu Event Generation
```
DefaultHomePageResponseSerializer.recordFallbackAndEvents(homepageContext: ExploreContext, data)
  -> LegacyHomePageIguazuEventUtil.generateHomePageFeedSegmentEvents(homepageContext: ExploreContext, data)
    -> builds partial event from ExploreContext fields
      -> ContainerEventsGenerator(facet, partial: GeneratedMessageV3, ...)
```

Note: `ContainerEventsGenerator` does NOT receive ExploreContext directly. It receives a pre built partial proto. This is fine because signals are flushed into the logging map at the adapter level (which has ExploreContext), and ContainerEventsGenerator reads from the logging map.

## Score Modifier Tracing

Every value that lands in the `SCORE_MODIFIERS` Snowflake column was traced to its computation and logging site.

### Store Level Scores (on StoreEntity, logged in StoreCarouselDataAdapter)

| Score | Set in | Method | Context at computation | Context at logging |
|---|---|---|---|---|
| `predictionScore` | `HomepageProgrammaticProductServiceUtil.kt:215` | `updateStoreEntity()` | Yes (ExploreContext reachable) | Yes (ExploreContext param) |
| `srMultiplier` | `HomepageProgrammaticProductServiceUtil.kt:219` | `updateStoreEntity()` | Yes | Yes |
| `srUncertaintyScore` | `HomepageProgrammaticProductServiceUtil.kt:221` | `updateStoreEntity()` | Yes | Yes |
| `urMultiplier` | `HomepageProgrammaticProductServiceUtil.kt:220` | `updateStoreEntity()` | Yes | Yes |
| `expectedCommission` | `HomepageProgrammaticProductServiceUtil.kt:224` | `updateStoreEntity()` | Yes | Yes |
| `expectedDasherCost` | `HomepageProgrammaticProductServiceUtil.kt:223` | `updateStoreEntity()` | Yes | Yes |
| `genaiEmbeddingSimilarityScore` | `HomepageProgrammaticProductServiceUtil.kt:225` | `updateStoreEntity()` | Yes | Yes |
| `cuisineAffinitySimilarity` | `DiscoveryRankingHydrater.kt` | `rankingHydration(context: BaseDiscoveryProductContext)` | Yes (direct param) | Yes |

### Carousel Level Scores (on StoreCarousel, logged in StoreCarouselDataAdapterUtil)

| Score | Read from | Adapter method | Context at logging |
|---|---|---|---|
| `programmaticBoostMultiplier` | `storeCarousel.programmaticBoostMultiplier` | `StoreCarouselDataAdapterUtil.generateLogging(storeCarousel, context: ExploreContext, ...)` | Yes |
| `ucbUncertaintyScore` | `storeCarousel.ucbUncertaintyScore` | Same | Yes |
| `calibrationMultiplier` | `storeCarousel.calibrationMultiplier` | Same | Yes |
| `verticalIntentMultiplier` | `storeCarousel.verticalIntentMultiplier` | Same | Yes |
| `verticalBoostWeight` | `storeCarousel.verticalBoostWeight` | Same | Yes |
| `diversityScore` | `storeCarousel.diversityScore` | Same | Yes |
| `userIntentPredictionScore` | `storeCarousel.userIntentPredictionScore` | Same | Yes |

These carousel level fields originate from `ScoreWithFactors` objects built during the ranking phase inside `CollectionScorer` and `EntityRankerConfiguration`, which receive ExploreContext via `rankAndMergeContent(context: ExploreContext, content)`.

### Scoring Layer (Deepest Computation)

| Scorer | Context type received | ExploreContext reachable? |
|---|---|---|
| `DefaultHomePagePostProcessor.rankAndMergeContent()` | `ExploreContext` directly | Yes |
| `CollectionScorer.updateCollection()` | Generic `C` bound to `BaseDiscoveryProductContext` | Yes via interface default |
| `BaseScorer.score()` | Generic `C` bound to context type | Varies |
| `ContextualStoreScorer.score()` | `ContextualStoreScorerContext` | No (narrow context) |
| `SibylRegressor.computeRegressionScores()` | `RegressionContext` | No (narrow context) |
| `GeneratedRecommendationCarouselService.reRankStoresByScoreAndSimilarity()` | `BaseDiscoveryProductContext` | Yes via interface default |
| `DiscoveryRankingHydrater.rankingHydration()` | `BaseDiscoveryProductContext` | Yes via interface default |

The deepest scorers (`ContextualStoreScorer`, `SibylRegressor`) use narrow context types that do not include ExploreContext. This is expected and correct. These are pure computation functions. The caller that invoked the scorer and has pipeline context is the right place to record signals.

## Conclusion

ExploreContext (via `BaseDiscoveryProductContext` with a no op default) is available at:

1. Every place where scores are currently set on `StoreEntity` or `StoreCarousel`
2. Every adapter method that builds the logging map
3. Every pipeline orchestration method that invokes scorers

The only places it is NOT available are inside the deepest scorer implementations (`ContextualStoreScorer.score()`, `SibylRegressor.computeRegressionScores()`). These methods use narrow, scorer specific context types. This is not a gap because the pattern is: the scorer computes, the caller (which has pipeline context) records. This separation is correct and encouraged.
