# Horizontal Ranking Analysis

> Where within-carousel store ordering happens in the feed-service pipeline.

---

## Where It Fits

```
retrieval → grouping → [HORIZONTAL RANKING] → decoration → layout → post-processing
```

Horizontal ranking is done in **two parallel jobs** in `DefaultHomePagePipeline.kt`, both running after `restaurantStoreGroupingJob` and `restaurantStoreDecorationForRankingJob`:

| Job | Source | What it does |
|---|---|---|
| `restaurantStoreRankingJob` (line 138) | `DefaultHomePageStoreRanker.rank()` | All non-campaign carousels |
| `restaurantCampaignRankingJob` (line 160) | `DefaultHomePageCampaignRanker.rank()` | Deals/Offers For You campaign carousels |

Both complete before `restaurantStoreDecorationForUserExperienceJob` which runs S1/S2/S3 decoration reranking.

---

## Phase A: Scoring (parallel, before sort)

`DefaultHomePageStoreRanker.scoredCollections()` splits collections into 5 groups and scores them concurrently:

```
Collections
   │
   ├─── CONTEXTUAL_TASTE_CAROUSEL            → contextualStoreScorer.score()  (taste dish context)
   ├─── SECONDARY_CONTEXTUAL_TASTE_CAROUSEL  → contextualStoreScorer.score()  (secondary taste)
   ├─── REALTIME_CONTEXTUAL_TASTE_CAROUSEL   → contextualStoreScorer.score()  (realtime taste)
   ├─── RETENTION_RECOMMENDED_CAROUSEL       → retentionStoreScorer.score()
   └─── everything else                      → storeCollectionScorer.scoreCollections()
                                                (batches by RankingType, calls Sibyl per batch)
```

All 5 calls run via `coroutineScope { listOf(async{}, async{}, ...).awaitAll() }`.

### StoreCollectionScorer internals

`scoreCollections()` (`StoreCollectionScorer.kt`):
- Reads DV-keyed model config: `P13nRuntimeUtil.getDVTreatmentSibylModelMappingConfig(context.pageType)`
- Builds a `RegressionContext` per `(rankingType, modelName)` pair — collections sharing the same context are batched into one Sibyl RPC call
- Builds `PredictionFeatureSet` per store: entity IDs, ~30 numerical features (delivery time, distance, rating, price, etc.), categorical features (vertical, cuisine tags)
- Optional `getScoreModifierMap()` for per-business/store score multipliers

Scores are stored in `LiteStoreCollection.storePredictionScoresMap[storeId].finalScore`.

---

## Phase B: Sorting (dispatch on RankingType)

`DefaultHomePageStoreRanker.modifyLiteStoreCollection()` first creates `storesByScore` (sorted by `finalScore DESC`, tie-broken by `popularityScore`), then dispatches:

```kotlin
when (collection.rankingType) {
    CAROUSEL, NEW_VERTICAL_CAROUSEL,
    CONTEXTUAL_TASTE_CAROUSEL,
    INFLATION_MULTIPLIER_CAROUSEL,
    RETENTION_RECOMMENDED_CAROUSEL     → sortStoreEntitiesForCarousels()   ← 5-level comparator
    MX_LOGO_CAROUSEL                   → sortStoreEntitiesForMXLogoCarousels() ← vertical interleave
    PICKUP_STORE_FEED, MAP_CAROUSEL    → sortStoreEntitiesByPickupAvailableAndDistanceToConsumer()
    BOOKMARKS                          → SavedStoresService.sortBookmarkedStores()
    DISTANCE_DECAYED_POPULARITY,
    CATEGORY_AND_DISTANCE_DECAYED_POPULARITY → PickupStoreRanking.sortStores()
    WHOLE_ORDER_REORDER                → sorted by recency index of previous orders
    NOOP                               → no-op, return stores as-is
    else                               → StoreListService.sortedByStatus() or sortedHomepageStoreFeedByStatus()
                                         (AlwaysOpen/VLP path)
}
```

### The 5-Level Comparator Chain (Standard CAROUSEL)

`StoreCarouselService.sortDiscoveryStoresWithBizRules()` chains comparators in descending priority:

```
1. StoreStatusComparator    → Available > Temporarily Closed > Permanently Closed
2. ShipAnywhereComparator   → ShipAnywhere stores floated/sunk per experiment
3. StoreSortOrderMapComparator (campaign priority) → hardcoded pinned biz IDs from campaign sort config
4. StoreSortOrderMapComparator (manual sort order) → per-carousel manual overrides from storeCarouselMapping
5. Sibyl ML score           → sortedByScoreList index (lower index = higher score)
```

First comparator that returns non-zero wins. Sibyl score is the tiebreaker of last resort.

### MX Logo Carousel Vertical Interleaving

`sortStoreEntitiesForMXLogoCarousels()` runs the 5-level sort first, then picks the top store from each vertical and moves them to the front:

```
Before: [A1, A2, A3, B1, B2, B3, C1, C2, C3]  (A/B/C = vertical)
After:  [A1, B1, C1, A2, A3, B2, B3, C2, C3]
```

---

## Campaign Ranker (Separate Path)

`DefaultHomePageCampaignRanker` handles Deals / Offers For You carousels:

- Checks NV filter DV (`should_filter_nv_stores_from_campaign_ranker`) — removes NV stores from campaign results
- Reads pre-scored campaign data from **CRDB cache** (not live Sibyl call) via `restaurantStoreDecorationForRankingJob`
- Delegates to `OffersForYouCampaignRanker` for the actual sort
- The Deals carousel vertical `sortOrder` is set here, not in post-processing

---

## The S1/S2/S3 Blind Spot

After `restaurantStoreRankingJob` completes, `restaurantStoreDecorationForUserExperienceJob` runs decoration for user experience. **This includes S1/S2/S3 reranking inside `rerankDecoratedEntities()`.**

Critical: **Iguazu fires before this job runs.** The logging events record horizontal positions as they were after `restaurantStoreRankingJob`, not after S1/S2/S3. Users see S1/S2/S3 positions; Iguazu logs organic positions.

```
rankingJob → Iguazu fires → decorationForUserExperience (S1/S2/S3 rerank) → layout
                ↑                                          ↑
         logged position                            actual position
```

**This is the single highest-value fix for UBP horizontal observability.** Moving S1/S2/S3 inside the ranking job (or emitting a separate corrected Iguazu event post-decoration) would close this gap.

---

## Pipeline Dependency Graph

```
retrieval
   │
grouping
   │
   ├──── decorationForRanking (Sibyl features prefetch)
   │          │
   │    ┌─────┴──────────────────┐
   │    │                        │
   │  storeRankingJob        campaignRankingJob
   │    │                        │
   │    └────────────┬───────────┘
   │                 │
   │         decorationForUserExperience  ← S1/S2/S3 rerank (Iguazu fires BEFORE this)
   │                 │
   │           layoutProcessing
   │                 │
   └─── postProcessing (vertical ranking) + adsBlending (parallel)
```

---

## UBP Implications

| Problem | Impact | UBP Fix |
|---|---|---|
| S1/S2/S3 runs after Iguazu | Logged horizontal positions ≠ positions users see | Move S1/S2/S3 into ranking job or emit corrected event |
| Model name comes from scattered DV keys | Changing horizontal model requires multi-team DV coordination | Single experiment JSON declares model name per step |
| 5-level comparator hardcoded | Adding/removing a sort criterion requires code change | `HORIZONTAL_SORT` processor with `steps` in config |
| Taste scorer uses separate code path | Different tracing, different observability, harder to experiment on | Unified `HorizontalProcessor` interface collapses all scorer types |
| 30+ RankingType branches | Each type has subtly different behavior, hard to reason about | Config-driven per-carousel pipeline replaces the `when` block |

---

## Key Files

| File | What |
|---|---|
| `pipelines/homepage/.../DefaultHomePagePipeline.kt` | Job DAG: lines 138 (`storeRankingJob`), 160 (`campaignRankingJob`), 178 (decoration+S1/S2/S3) |
| `libraries/common/.../DefaultHomePageStoreRanker.kt` | `rank()` orchestrator + `modifyLiteStoreCollection()` dispatch |
| `libraries/domain-util/.../StoreCollectionScorer.kt` | Sibyl batch scoring for most carousel types |
| `libraries/domain-util/.../ContextualStoreScorer.kt` | Taste carousel scorer |
| `libraries/domain-util/.../RetentionStoreScorer.kt` | Retention carousel scorer |
| `libraries/domain-util/.../StoreCarouselService.kt` | `sortDiscoveryStoresWithBizRules()` 5-level comparator, MX logo interleaving |
| `libraries/domain-util/.../DefaultHomePageCampaignRanker.kt` | Deals/Offers For You ranker |
| `libraries/sdk/.../RankingType.kt` | All 30+ RankingType values |
