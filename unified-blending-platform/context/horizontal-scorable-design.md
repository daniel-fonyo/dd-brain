# Horizontal Ranking: Scorable Interface Design

> How to make within-carousel elements implement `Scorable` and rank them via the UBP engine.

---

## Current State

### Shipped (Phase 1 — vertical)
- `Scorable` interface: `scorableId()`, `predictionScore`, `withPredictionScore()`
- `Ranker<S>` engine with chain-of-responsibility handlers
- `VerticalStepType { RANK_ALL }` + `VerticalRankAllStep` wrapping legacy `RankerConfiguration.rank()`
- 9 carousel types implement `Scorable` (StoreCarousel, ItemCarousel, DealCarousel, etc.)
- Shadow validation: 99% match on sandbox

### Not shipped
- No horizontal (intra-carousel) UBP code
- No `IntraCarouselRankStepType`
- `DiscoveryStore` does not implement `Scorable`

---

## The Horizontal Element Types

Within-carousel ranking operates on these types:

| Type | Where ranked | Scoring source | Current ranking method |
|---|---|---|---|
| **DiscoveryStore** | `modifyLiteStoreCollection()` | `LiteStoreCollection.storePredictionScoresMap[storeId].finalScore` | 5-level comparator chain via `sortDiscoveryStoresWithBizRules()` |
| **ItemStoreEntity** | Item carousel paths | Item-level prediction scores | Various item carousel rankers |

`DiscoveryStore` is the primary type — it covers all standard carousels, contextual taste, retention, MX logo, pickup, bookmarks, etc. (the full `RankingType` when-block).

---

## The Challenge

`DiscoveryStore` differs from the vertical carousel types in three ways:

### 1. Score is external, not on the object
Vertical types carry their score: `StoreCarousel.predictionScore`. But `DiscoveryStore` has no `predictionScore` field — the score lives in `LiteStoreCollection.storePredictionScoresMap[storeId].finalScore`, looked up by ID during sorting.

**Solution:** Add `predictionScore: Double?` field to `DiscoveryStore`. Hydrate it from the map before passing to the ranker. This is a one-time transformation, same pattern as the vertical `toScorableList()`.

### 2. ID type mismatch
`DiscoveryStore.id` is `Long`. `Scorable.scorableId()` returns `String`.

**Solution:** `override fun scorableId(): String = id.toString()` — same approach used by `StoreEntity` in the plan.

### 3. Ranking isn't just score-based
The 5-level comparator chain applies business rules (availability, ship-anywhere, campaign priority, manual sort) before falling back to ML score. This is fundamentally different from vertical ranking which is primarily score-driven.

**Solution:** The `IntraCarouselRankAllStep` wraps the entire `modifyLiteStoreCollection()` logic, including all comparators. The business rules are part of the step — they don't need to be score-based to go through the engine. Phase 2 can decompose them into separate steps (e.g., `AVAILABILITY_SORT`, `BUSINESS_PRIORITY`, `ML_SCORE`).

---

## Design

### DiscoveryStore implements Scorable

```kotlin
data class DiscoveryStore(
    override val id: Long,
    // ... existing fields ...
    override val predictionScore: Double? = null,  // NEW: hydrated from collection's score map
) : SdkEntity, Scorable {

    override fun scorableId(): String = id.toString()

    override fun withPredictionScore(score: Double): DiscoveryStore =
        copy(predictionScore = score)
}
```

### Score Hydration

Before ranking, hydrate scores from the collection's map onto the DiscoveryStore objects:

```kotlin
fun List<DiscoveryStore>.hydratePredictionScores(
    scoreMap: Map<Long, ScoreWithFactors>
): List<DiscoveryStore> = map { store ->
    val score = scoreMap[store.id]?.finalScore
    if (score != null) store.withPredictionScore(score) as DiscoveryStore else store
}
```

### IntraCarouselRankStepType

```kotlin
enum class IntraCarouselRankStepType {
    RANK_ALL,
    // Phase 2: AVAILABILITY_SORT, BUSINESS_PRIORITY, CAMPAIGN_SORT, ML_SCORE
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
        items: List<Scorable>,
        context: RankingContext
    ): List<Scorable> {
        // Cast items to DiscoveryStore, apply the existing sorting logic,
        // return in new order. Exact delegation TBD based on what context
        // is available (RankingType, collection metadata, etc.)
    }
}
```

### Wiring

In `DefaultHomePageStoreRanker.rank()`, wrap the per-collection ranking:

```kotlin
// Before (current):
val rankedCollection = modifyLiteStoreCollection(context, collection, storesById, metadata, ...)

// After (UBP):
val stores = collection.storeIds.mapNotNull(storesById::get)
    .hydratePredictionScores(collection.storePredictionScoresMap)
val ranked = intraCarouselRanker.rank(
    items = stores,
    stepTypes = listOf(IntraCarouselRankStepType.RANK_ALL),
    context = rankingContext
)
```

---

## Key Decision: What does `IntraCarouselRankAllStep` wrap?

Two options:

### Option A: Wrap `modifyLiteStoreCollection()` (the dispatch)
- Step receives the full collection context (RankingType, metadata)
- Delegates to the existing when-block internally
- Pro: Captures ALL horizontal ranking paths (standard, pickup, bookmarks, noop, etc.)
- Con: Step needs collection-level context beyond what `RankingContext` provides

### Option B: Wrap `sortDiscoveryStoresWithBizRules()` (the 5-level chain only)
- Step only handles the standard CAROUSEL path
- Other RankingTypes (PICKUP, BOOKMARKS, NOOP) stay outside the engine
- Pro: Cleaner — step only does score-based+business-rule sorting
- Con: Doesn't capture the full horizontal ranking surface

**Recommendation: Option A** — wrap the entire `modifyLiteStoreCollection()`. This gives UBP coverage over ALL horizontal ranking paths. The RankingType dispatch is part of the ranking logic and belongs in the engine. Non-standard paths (NOOP, BOOKMARKS) are still "ranking decisions" — they just happen to be trivial ones.

The collection-level context (RankingType, campaigns, manual sort orders) can be passed via an extended `RankingContext` or a separate `IntraCarouselContext` wrapper.

---

## Phase 2 Decomposition Preview

Once `RANK_ALL` is stable, decompose into:

```
IntraCarouselRankStepType {
    RANK_ALL,           // Phase 1: wraps everything
    SCORE_HYDRATION,    // Hydrate scores from collection map onto DiscoveryStore
    AVAILABILITY_SORT,  // StoreStatusComparator
    SHIP_ANYWHERE,      // ShipAnywhereComparator
    BUSINESS_PRIORITY,  // Campaign priority + manual sort
    ML_SCORE,           // Prediction score tiebreak
    MX_LOGO_INTERLEAVE, // Vertical interleaving for MX logo carousels
}
```

Each step is independently testable and swappable — e.g., an experiment can replace `ML_SCORE` with a different scorer without touching availability or business rules.

---

## Open Questions

1. **DiscoveryStore field addition**: Adding `predictionScore: Double?` to DiscoveryStore means all callers that construct it need updating (or we use a default). Since it's a data class, `= null` default should handle this cleanly.

2. **Context threading**: `modifyLiteStoreCollection()` uses `ExploreContext` (which extends `RankingContext`). The UBP engine uses `RankingContext`. Need to verify the existing `RankingContext` has everything the step needs, or extend it.

3. **Collection-level vs store-level**: The engine operates on `List<Scorable>` (stores). But `modifyLiteStoreCollection()` also modifies the collection itself (updates `storeIds`, filters `storePredictionScoresMap`). The step needs to handle this reconstruction.

4. **Campaign ranker**: `DefaultHomePageCampaignRanker` is a separate path from `DefaultHomePageStoreRanker`. Should it go through the same `IntraCarouselRankAllStep`, or get its own step?
