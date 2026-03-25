# Intra-Carousel Ranking: Rankable Interface Design

> How to make within-carousel elements implement `Rankable` and rank them via the UBP engine.
>
> **Naming source of truth:** `spec/rfc-abstraction-layer.md`

---

## Current State

### Shipped (Phase 1 — inter-carousel, POC)
- `Rankable` interface: `rankableId()`, `predictionScore`, `withPredictionScore()`
- `RankingPipeline<S>` engine with chain-of-responsibility `StepHandler`s
- `CarouselRankStepType { RANK_ALL }` + `CarouselRankAllStep` wrapping legacy `RankerConfiguration.rank()`
- 9 carousel types implement `Rankable` (StoreCarousel, ItemCarousel, DealCarousel, etc.)
- Shadow validation: 99% match on sandbox
- POC branch uses stale names (`Scorable`, `Ranker`, `VerticalStepType`) — will be renamed

### Not shipped
- No intra-carousel UBP code exists yet
- `DiscoveryStore` does not implement `Rankable`
- No `IntraCarouselRankStepType`

---

## The Intra-Carousel Element Types

Within-carousel ranking operates on these types:

| Type | Where ranked | Scoring source | Current ranking method |
|---|---|---|---|
| **DiscoveryStore** | `modifyLiteStoreCollection()` | `LiteStoreCollection.storePredictionScoresMap[storeId].finalScore` | 5-level comparator chain via `sortDiscoveryStoresWithBizRules()` |
| **ItemStoreEntity** | Item carousel paths | Item-level prediction scores | Various item carousel rankers |

`DiscoveryStore` is the primary type — covers all standard carousels, contextual taste, retention, MX logo, pickup, bookmarks, etc. (the full `RankingType` when-block in `modifyLiteStoreCollection()`).

---

## Three Gaps to Bridge

### 1. Score is external, not on the object

Inter-carousel types carry their score: `StoreCarousel.predictionScore`. But `DiscoveryStore` has no `predictionScore` field — the score lives in `LiteStoreCollection.storePredictionScoresMap[storeId].finalScore`, looked up by ID during sorting.

**Solution:** Add `predictionScore: Double? = null` field to `DiscoveryStore`. Hydrate from the map before passing to the ranking pipeline. Same principle as `toRankableList()` for inter-carousel ranking.

```kotlin
fun List<DiscoveryStore>.hydratePredictionScores(
    scoreMap: Map<Long, ScoreWithFactors>
): List<DiscoveryStore> = map { store ->
    val score = scoreMap[store.id]?.finalScore
    if (score != null) store.withPredictionScore(score) as DiscoveryStore else store
}
```

### 2. ID type mismatch

`DiscoveryStore.id` is `Long`. `Rankable.rankableId()` returns `String`.

**Solution:** `override fun rankableId(): String = id.toString()`

### 3. Ranking isn't just score-based

The 5-level comparator chain applies business rules (availability, ship-anywhere, campaign priority, manual sort) before falling back to ML score. Fundamentally different from inter-carousel ranking which is primarily score-driven.

**Solution:** The `IntraCarouselRankAllStep` wraps the entire `modifyLiteStoreCollection()` logic, including all comparators and the `RankingType` dispatch. The business rules are part of the step. Phase 2 decomposes them into separate steps (`AVAILABILITY_SORT`, `BUSINESS_PRIORITY`, `ML_SCORE`).

---

## Design

### DiscoveryStore implements Rankable

```kotlin
data class DiscoveryStore(
    override val id: Long,
    // ... existing fields ...
    override val predictionScore: Double? = null,  // NEW: hydrated from collection score map
) : SdkEntity, Rankable {

    override fun rankableId(): String = id.toString()

    override fun withPredictionScore(score: Double): DiscoveryStore =
        copy(predictionScore = score)
}
```

### IntraCarouselRankStepType

```kotlin
enum class IntraCarouselRankStepType {
    RANK_ALL,
    // Phase 2: AVAILABILITY_SORT, BUSINESS_PRIORITY, CAMPAIGN_SORT, ML_SCORE, MX_LOGO_INTERLEAVE
}
```

### IntraCarouselRankAllStep

Wraps the entire existing `modifyLiteStoreCollection()` dispatch logic:

```kotlin
class IntraCarouselRankAllStep(
    private val storeCarouselService: StoreCarouselService,
    // other dependencies needed by the when-block
) : RankingStep<IntraCarouselRankStepType> {
    override val stepType = IntraCarouselRankStepType.RANK_ALL

    override suspend fun execute(
        items: List<Rankable>,
        context: RankingContext
    ): List<Rankable> {
        // Cast items to DiscoveryStore, apply the existing sorting logic,
        // return in new order. Delegates to the same modifyLiteStoreCollection()
        // methods the current path calls.
    }
}
```

### Wiring into DefaultHomePageStoreRanker

In `DefaultHomePageStoreRanker.rank()`, wrap the per-collection ranking:

```kotlin
// Before (current):
val rankedCollection = modifyLiteStoreCollection(context, collection, storesById, metadata, ...)

// After (UBP):
val stores = collection.storeIds.mapNotNull(storesById::get)
    .hydratePredictionScores(collection.storePredictionScoresMap)
val ranked = intraCarouselPipeline.rank(
    items = stores,
    stepTypes = listOf(IntraCarouselRankStepType.RANK_ALL),
    context = rankingContext
)
// Reconstruct collection with new store order
```

### RankingPipeline instantiation

```kotlin
val intraCarouselPipeline = RankingPipeline(
    stepRegistry = mapOf(
        IntraCarouselRankStepType.RANK_ALL to IntraCarouselRankAllStep(storeCarouselService, ...)
    )
)
```

---

## Key Decision: What does IntraCarouselRankAllStep wrap?

**Option A: Wrap `modifyLiteStoreCollection()` (the full dispatch)** — RECOMMENDED
- Step receives the full collection context (RankingType, metadata)
- Delegates to the existing when-block internally
- Pro: Captures ALL intra-carousel ranking paths (standard, pickup, bookmarks, noop, etc.)
- Con: Step needs collection-level context beyond what `RankingContext` provides

**Option B: Wrap `sortDiscoveryStoresWithBizRules()` (the 5-level chain only)**
- Step only handles the standard CAROUSEL path
- Other RankingTypes (PICKUP, BOOKMARKS, NOOP) stay outside the engine
- Pro: Cleaner — step only does score-based+business-rule sorting
- Con: Doesn't capture the full intra-carousel ranking surface

**Choice: Option A.** The RankingType dispatch is ranking logic and belongs in the engine. Non-standard paths (NOOP, BOOKMARKS) are still ranking decisions — they just happen to be trivial ones. The collection-level context (RankingType, campaigns, manual sort orders) can be passed via an extended `RankingContext` or a `IntraCarouselContext` added to the context.

---

## Phase 2 Decomposition Preview

Once `RANK_ALL` is stable, decompose into granular steps:

```
IntraCarouselRankStepType {
    RANK_ALL,           // Phase 1.5: wraps everything
    SCORE_HYDRATION,    // Hydrate scores from collection map onto DiscoveryStore
    AVAILABILITY_SORT,  // StoreStatusComparator
    SHIP_ANYWHERE,      // ShipAnywhereComparator
    BUSINESS_PRIORITY,  // Campaign priority + manual sort
    ML_SCORE,           // Prediction score tiebreak
    MX_LOGO_INTERLEAVE, // Vertical interleaving for MX logo carousels
}
```

Each step independently testable and swappable — e.g., an experiment can replace `ML_SCORE` with a different scorer without touching availability or business rules.

---

## Open Questions

1. **DiscoveryStore field addition**: Adding `predictionScore: Double? = null` to DiscoveryStore means all callers that construct it need updating (or we use a default). Since it's a data class, `= null` default handles this cleanly.

2. **Context threading**: `modifyLiteStoreCollection()` uses `ExploreContext`. The UBP engine uses `RankingContext`. Need to verify `RankingContext` has everything the step needs, or extend it.

3. **Collection-level vs store-level**: The engine operates on `List<Rankable>` (stores). But `modifyLiteStoreCollection()` also modifies the collection itself (updates `storeIds`, filters `storePredictionScoresMap`). The step needs to handle this reconstruction, or the reconstruction happens after the step returns.

4. **Campaign ranker**: `DefaultHomePageCampaignRanker` is a separate path. Should it go through the same `IntraCarouselRankAllStep`, or get its own step?
