# Horizontal Ranking Deep Dive: How Within-Carousel Store Ranking Works

> Source: `DefaultHomePageStoreRanker.kt`, `DefaultHomePageCampaignRanker.kt`,
> `StoreCollectionScorer.kt`, `StoreCarouselService.kt`
>
> Horizontal ranking happens in the pipeline BEFORE post-processing.
> It determines the order of stores/items within each carousel (slot 0, slot 1, slot 2...).
> Post-processing (vertical ranking) handles which carousel row goes first — that's separate.

---

## Where It Lives in the Pipeline

```
restaurantStoreGroupingJob          ← assigns stores to carousels
         │
         ▼
restaurantStoreRankingJob           ← horizontal ranking for ALL non-campaign carousels
restaurantCampaignRankingJob        ← horizontal ranking for Offers For You / Deal carousel (separate parallel job)
         │
         ▼
restaurantStoreDecorationForUserExperienceJob   ← decoration + S1/S2/S3 reranking (post-ranking override — covered below)
         │
         ▼
restaurantStoreLayoutProcessingJob  ← constructs HomePageStoreLayoutOutputElements
         │
         ▼
postProcessingJob                   ← vertical ranking (separate doc)
```

**Two separate rankers run in parallel:**
- `homePageRestaurantStoreRanker` → scores and sorts stores in all programmatic/taste/NV carousels
- `homePageCampaignRanker` → scores and sorts stores for the Deal / Offers For You carousel only

---

## Part 1: Non-Campaign Horizontal Ranking (DefaultHomePageStoreRanker)

### Phase A: Score All Stores (scoredCollections)

Before any sorting, ALL stores across ALL collections get ML scores from Sibyl.
Collections are partitioned by scoring type because each uses a different model/context:

```
collections
    │
    ├─ CONTEXTUAL_TASTE_CAROUSEL          ─▶ contextualStoreScorer.score()
    │    (e.g., "Pizza", "Sushi" taste)        uses: taste query string as scoreContext
    │                                           DV: SpecificTasteV2Carousel.getTaste()
    │
    ├─ SECONDARY_CONTEXTUAL_TASTE_CAROUSEL ─▶ contextualStoreScorer.score()
    │    (secondary taste, e.g. "Burgers")      uses: secondary taste query string
    │                                           DV: SecondarySpecificTasteV2Carousel.getTaste()
    │
    ├─ REALTIME_CONTEXTUAL_TASTE_CAROUSEL  ─▶ contextualStoreScorer.score()
    │    (realtime taste signal)                uses: realtime taste string
    │                                           DV: RealTimeSpecificTasteCarousel.getTaste()
    │
    ├─ RETENTION_RECOMMENDED_CAROUSEL      ─▶ retentionStoreScorer.score()
    │    (churn/retention ML)                   uses: RetentionStoreScorerContext
    │
    └─ everything else                     ─▶ storeCollectionScorer.scoreCollections()
         (CAROUSEL, NEW_VERTICAL_CAROUSEL,      uses: standard regression context
          INFLATION_MULTIPLIER_CAROUSEL,        model: from DVTreatmentSibylModelMappingConfig
          STORE_FEED, store lists)              per rankingType label
```

All of these fire **in parallel via coroutines** (`awaitAll()`). One Sibyl batch call
per scorer type, not one call per carousel.

**NOOP carousels skip scoring entirely** — their stores keep whatever order they came in.

### The Scoring Mechanics (StoreCollectionScorer)

For each store, Sibyl receives a `PredictionFeatureSet` containing:
```
entity IDs:
  consumer_id, store_id, business_id, business_vertical_id, experience_id, request_id

numerical features (store-level):
  distance, asap_minutes, delivery_fee, price_range, star_rating, num_ratings,
  seconds_to_closing, popularity_score, is_dashpass_partner, ...

categorical features:
  day_of_week, day_part, platform (ios/android/web), ...

list features (when DV enabled):
  surface-aware features (ListFeature) — enabled via:
    shouldUseSRFWPredictorForStoreRanker DV
    shouldUseNVCarouselFWPredictor DV
    shouldUseNVCarouselSurfaceAwareFeatures DV
```

**Model selection per collection type** — runtime JSON `DVTreatmentSibylModelMappingConfig`:
```
{
  "carousel":                     { "model": "store_ranking_lr15_v2" },
  "new_vertical_carousel":        { "model": "nv_store_ranker_v3" },
  "contextual_taste_carousel":    { "model": "taste_ranker_v2" },
  "retention_recommended_carousel": { "model": "retention_ranker_v1" },
  ...
}
```

Multiple carousels with the SAME ranking type share one Sibyl call (they're batched by
`RegressionContext.key()`). Distinct types each get their own Sibyl call.

**Fallback:** If Sibyl returns no score for a store → use `store.realtimeFeatures.popularityScore`.

**Score modifier (optional):** After Sibyl, a per-business or per-store multiplier can be applied:
```
finalScore = sibylScore × scoreModifierMap[storeId or businessId]
```
Multipliers are defined in `PersonalizationRuntimeUtil.getStoreRankerBoostMultipliersByBusiness()`.
Only active when `shouldBoostBusinessForStoreRanker` DV is on.

**Multi-label models:** Some models return separate scores per label (CTR, GOV, retention).
These are combined into `finalScore` by the model itself (Sibyl handles the aggregation).
Individual label scores stored in `ScoreWithFactors.multiLabelScores` for logging.

---

### Phase B: Sort Stores Within Each Carousel (modifyLiteStoreCollection)

After scoring, each collection is sorted **in parallel** (one coroutine per carousel).

```
STEP 1: Sort by ML score (always applied first, regardless of ranking type)

  storesByScore = stores.sortedWith(
    compareByDescending { collection.storePredictionScoresMap[it.id]?.finalScore ?: 0.0 }
    .thenByDescending { it.realtimeFeatures.popularityScore }   ← tie-breaker
  )
```

Then a `when (collection.rankingType)` applies type-specific additional logic on top:

---

#### Ranking Type: `CAROUSEL`, `NEW_VERTICAL_CAROUSEL`, `CONTEXTUAL_TASTE_CAROUSEL`, `INFLATION_MULTIPLIER_CAROUSEL`, `RETENTION_RECOMMENDED_CAROUSEL`

All use the same multi-level comparator chain:

```
sortDiscoveryStoresWithBizRules()

Priority order (earlier = higher priority, later = tiebreaker):

  1. StoreStatusComparator       ← available stores first, unavailable last
     │  available = asapAvailable OR pickupAvailable OR alwaysOpen (NV schedule-ahead)
     │  AlwaysOpenType DV controls whether NV stores count as "open"

  2. ShipAnywhereComparator      ← only if carouselId is in ShipAnywhere allowlist
     │  Ship-anywhere eligible stores ranked above non-eligible stores within each status group

  3. StoreListComparator(prioritizedStoreIds)   ← specific stores forced to top
     │  Source: businessSortOrderOverride map, keyed by carouselId
     │  Typically used for national/flagship chains that pay for priority placement

  4. StoreSortOrderMapComparator(storeIdSortOrderMap)   ← manual sort order from retrieval data
     │  Source: homePageContent.storeCarouselMapping (passed in from retrieval as metadata)
     │  Overrides ML ranking. Stores with a manual sort order rank above those without one.
     │  null sort order → treated as lowest priority within this tier

  5. StoreListComparator(sortedByScoreList)   ← final tiebreaker: ML score order
     │  sortedByScoreList = storesByScore.map(DiscoveryStore::id) from STEP 1
```

**Reading this:** A store that is CLOSED is always shown after ALL open stores, regardless
of how high its ML score is. Within the open stores: prioritized business > manual sort order > ML score.

---

#### Ranking Type: `MX_LOGO_CAROUSEL`

Tall logo/brand carousel. First applies the standard `sortStoreEntitiesForCarousels()` rules above,
then adds a vertical interleaving step:

```
storesWithDefaultRanking = sortStoreEntitiesForCarousels(...)

Priority business IDs (from getMxLogoPriorityBizIds()):
  → these businesses always go to the front regardless of score

Vertical top stores (from getMxLogoPriorityVerticalIds()):
  → for each priority vertical (e.g., Grocery, NV), surface the top-ranked store for that vertical
  → these stores are pulled out and placed after priority stores but before everyone else

Final order: [priority businesses] + [1 top store per priority vertical] + [remaining by score]

Example:
  Before: [A1, A2, A3, B1, B2, B3, C1, C2, C3]  (A=Rx, B=Grocery, C=NV vertical top brands)
  After:  [A1, B1, C1, A2, A3, B2, B3, C2, C3]  ← 1 from each vertical at front
```

---

#### Ranking Type: `PICKUP_STORE_FEED`, `MAP_CAROUSEL`

ML score is completely ignored. Pure operational sort:
```
compareByDescending { it.pickupAvailable }
  .thenComparing(compareBy { it.realtimeFeatures.distance })
```
Pickup-available stores first, then closest first within that group.

---

#### Ranking Type: `BOOKMARKS`

ML score is completely ignored. Sort by bookmark metadata:
```
SavedStoresService.sortBookmarkedStores(
  savedStores = bookmarkedSavelistEntities,
  bookmarkedItems = bookmarkedItems,
  previouslyOrderedStores = previouslyOrderedStoreIds,
  consumerContext,
  experimentMap,
)
```
Priority: stores you've recently ordered from first, then stores you've saved, then remaining.
The sort logic is in `SavedStoresService` (separate service, not in the ranker).

---

#### Ranking Type: `WHOLE_ORDER_REORDER`

ML score is completely ignored. Sort by how often/recently you've reordered:
```
storeIdToPositionMap = WholeOrderReorderFilterUtil.getGoToOrders(previousOrders)
  .mapIndexed { index, order -> order.storeId to index }

stores.sortedBy { storeIdToPositionMap[it.id] ?: Int.MAX_VALUE }
```
Your most-reordered store goes first. Stores not in order history → end of list.

---

#### Ranking Type: `DISTANCE_DECAYED_POPULARITY`, `CATEGORY_AND_DISTANCE_DECAYED_POPULARITY`

```
PickupStoreRanking.sortStores(context, collection, stores)
```
Combines distance decay with popularity score. Closer + more popular = higher rank.
`CATEGORY_AND_DISTANCE_DECAYED_POPULARITY` additionally groups by category first.

---

#### Ranking Type: `NOOP`

No sorting. Returns stores in whatever order they came from grouping.

---

#### `else` branch: Store List and Vertical Page Sorts

Catch-all for store lists, vertical pages, and any unrecognized ranking types:

```
if (pageType == VERTICAL_PAGE) {
    if (verticalId in [CATERING, LARGE_ORDER_QUALITY] && isScheduledOrder) {
        StoreListService.sortedByStatus(storesByScore, isScheduledOrder)
        ← available first, then scheduled-available, then unavailable
    } else {
        StoreListService.sortVerticalPageStoreList(
            scoredStoreEntities = storesByScore,
            primaryVerticalId = verticalId,
            shouldPrioritizeFlagShipStores = isFlagshipEnabled && shouldEnableLogoMerchandising,
            alwaysOpenType = context.exploreLayoutContext.alwaysOpenType,
        )
        ← flagship stores to front if DV enabled, then available by score, then unavailable by score
    }
} else {
    ← homepage and other page types
    if (alwaysOpenRankingType == CONTROL || isScheduledOrder) {
        StoreListService.sortedByStatus(storesByScore, isScheduledOrder)
    } else {
        StoreListService.sortedHomepageStoreFeedByStatus(stores, alwaysOpenContext)
        ← alwaysOpen NV stores get mixed into available group, not pushed to available-last
    }
}
```

---

### Phase C: Pagination

After sorting, each collection is truncated:
```
paginatedStores = CommonStoreRankerUtil.paginate(sortedStores, collection.paginatedParams)
```
`paginatedParams.limit` defines how many stores the carousel keeps (typically 20-30).
Stores beyond this limit are dropped before layout processing ever sees them.

---

## Part 2: Campaign Horizontal Ranking (DefaultHomePageCampaignRanker)

The Deal / Offers For You carousel is ranked entirely separately via its own job.

```
INPUT: all DiscoveryStores (or filtered subset)
         │
         ├─ Filter: NV store filter (DV: DEAL_FOR_YOU_CAROUSEL_NV_FILTER)
         │     If enabled: remove stores where businessVerticalId != null && != 0
         │     UNLESS businessId is in NV allowlist (DV: DEAL_FOR_YOU_CAROUSEL_NV_ALLOW_LIST)
         │     → Result: Rx-only or Rx+allowlisted NV stores
         │
         ├─ Optional filter: DFY promo only (DV: shouldRankStoresByDfyPromoOnlyDataModel)
         │     If enabled: fetch campaign data from CRDB cache
         │     Keep only stores that have an active campaign
         │     Reduces Sibyl input from ~1000 stores to ~500-700
         │     Fallback: if no campaigns found → use all stores
         │
         └─ Build a synthetic LiteStoreCollection (id = "deal_carousel")
               rankingType = CAMPAIGN
               storeIds = stores from campaign data
               sortOrder = DealService.position(experimentMap)   ← vertical sort from DV
               carouselType = DEAL
         │
         ▼
offersForYouCampaignRanker.rank()
    │
    ├─ Data source: storeCampaignInfoCacheRepository.getStoreCampaignInfoMap()
    │     "New path": offline CRDB cache (campaign-specific features)
    │     vs old path (promotion service, now deprecated)
    │     Controlled by: shouldStoreCampaignUseNewCrdbSource DV
    │
    └─ Scores stores via Sibyl with campaign-specific features
       (deal discount %, campaign type, store promotion history)
       Output: CampaignRankingMetadataV2(newPath = rankedCollection)
```

**The deal carousel's vertical position** (`DealService.position(experimentMap)`) is set here
during horizontal ranking, not during post-processing. Post-processing then either pins it
into the pinned list or lets it be overridden by `shouldUpdateOffersForYouSortOrder`
(the random-within-top-5 logic described in the vertical ranking doc).

---

## Part 3: The Hidden S1/S2/S3 Reranking (After Horizontal Ranking)

This is the most important thing for UBP to know: there's a **second horizontal ranking step**
that happens in `restaurantStoreDecorationForUserExperienceJob`, AFTER the main ranking job,
in the **decoration** step. It's not in the ranker — it's buried in the decorator.

```
restaurantStoreDecorationForUserExperienceJob
  │
  homePageRestaurantExperienceDecorator.decorate()
  │
  └─ rerankDecoratedEntities()    ← THIS is S1/S2/S3
       │
       Applies final reranking to decorated stores
       Uses previously ordered stores, bookmarks, and experience-specific signals
       to adjust store positions within certain carousels AFTER scoring
```

**Why this matters:** The Iguazu logging event fires based on scores from `restaurantStoreRankingJob`.
But the actual horizontal position the user sees is determined AFTER `rerankDecoratedEntities()`
runs in decoration. So:
- Iguazu logs `horizontal_position_in_facet = 3` for store X
- The user actually sees store X at position 5 because S1/S2/S3 moved it

This is the horizontal ranking equivalent of the vertical position discrepancy problem.
S1/S2/S3 is completely unobservable in the current logging.

---

## Summary: What Determines Horizontal Position

For a standard `CAROUSEL` type (the most common), the priority stack is:

```
PRIORITY 1: Store availability (open/closed)
            Closed stores always last, regardless of score

PRIORITY 2: ShipAnywhere eligibility (carousel-specific, if enabled)

PRIORITY 3: Prioritized business IDs (merchandising, national favorites)
            Set via runtime config, not ML

PRIORITY 4: Manual sort order from storeCarouselMapping
            Set during retrieval via carousel config
            Overrides ML ranking. Promotional/sponsored ordering.

PRIORITY 5: Sibyl ML score (finalScore)
            The only thing that's learned from data
            Multiple models: standard, taste, NV, retention, contextual

PRIORITY 6: popularityScore (tiebreaker within same ML score bucket)

AFTER ALL OF THE ABOVE: S1/S2/S3 decoration reranking
            Not in logging. Not observable. Applied after Iguazu fires.
```

**Reading this:** ML score is priority 5 out of 6 before decoration. The user experience
is heavily governed by business rules, not the model.

---

## The 30+ Ranking Types At a Glance

| RankingType | Scoring method | Sort logic | ML used? |
|---|---|---|---|
| `CAROUSEL` | Sibyl (standard model) | availability → bizRules → manualSort → ML | ✅ |
| `NEW_VERTICAL_CAROUSEL` | Sibyl (NV model) | availability → bizRules → manualSort → ML | ✅ |
| `CONTEXTUAL_TASTE_CAROUSEL` | Sibyl (taste model + query) | availability → bizRules → manualSort → ML | ✅ |
| `SECONDARY_CONTEXTUAL_TASTE_CAROUSEL` | Sibyl (secondary taste) | availability → bizRules → manualSort → ML | ✅ |
| `REALTIME_CONTEXTUAL_TASTE_CAROUSEL` | Sibyl (realtime taste) | availability → bizRules → manualSort → ML | ✅ |
| `INFLATION_MULTIPLIER_CAROUSEL` | Sibyl (standard) | availability → bizRules → manualSort → ML | ✅ |
| `RETENTION_RECOMMENDED_CAROUSEL` | Sibyl (retention model) | availability → bizRules → manualSort → ML | ✅ |
| `MX_LOGO_CAROUSEL` | Sibyl (standard) | standard rules + vertical interleaving | ✅ |
| `CAMPAIGN` | Sibyl (campaign model) | campaign-specific, separate ranker | ✅ |
| `PICKUP_STORE_FEED` | None | pickup available → distance | ❌ |
| `MAP_CAROUSEL` | None | pickup available → distance | ❌ |
| `BOOKMARKS` | None | order history → bookmark recency | ❌ |
| `WHOLE_ORDER_REORDER` | None | reorder frequency/position | ❌ |
| `DISTANCE_DECAYED_POPULARITY` | None | distance × popularity decay | ❌ |
| `CATEGORY_AND_DISTANCE_DECAYED_POPULARITY` | None | category + distance × popularity decay | ❌ |
| `NOOP` | None | no change | ❌ |
| Store list (homepage) | Sibyl (standard) | alwaysOpen-aware status sort | ✅ |
| Store list (VLP) | Sibyl (standard) | flagship first (if DV), then status sort | ✅ |

---

## What This Means for UBP

### The core problem with horizontal ranking today:

1. **No shared interface across 30+ ranking types.** Each `RankingType` is a separate code
   branch with different rules, different models, different sort logic. Adding any cross-cutting
   concern (e.g., "log score before and after business rules") requires touching all 30+ branches.

2. **Business rules are hardcoded before the ML step is visible.**
   `StoreStatusComparator` runs first, always. You can't experiment with "what if open/closed
   didn't override ML score?" without changing the comparator.

3. **S1/S2/S3 is after the observable pipeline.** The Iguazu event fires with pre-decoration
   positions. The positions the user sees are post-decoration. There's no logging between
   `restaurantStoreRankingJob` and `rerankDecoratedEntities()`.

4. **Manual sort order silently overrides ML.** `storeCarouselMapping` is retrieved during
   content fetching and applied during ranking without any tracing. You can't tell from logs
   whether position 0 was the model or a manual override.

### What UBP does about it:

The `HorizontalProcessor` interface resolves all of these:
- `ModelScoringProcessor` → handles Sibyl call
- `BusinessRulesSortProcessor` → handles availability/manualSortOrder (declared, not hardcoded)
- `OrderHistoryRerankProcessor` → absorbs S1/S2/S3 into the observable pipeline
- All processor params injectable from config → experiments can change priority order without code

The S1/S2/S3 move into the pipeline is the highest-value single change in horizontal UBP,
because it makes the largest unobserved step fully visible.
