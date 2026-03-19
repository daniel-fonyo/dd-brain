# Part 2: FeedRow Interface + Adapters

## Goal

Define a single interface wrapping all 9 carousel domain types so ranking steps operate on
`MutableList<FeedRow>` and never branch on type. Adapters are the translation layer between
the existing domain model and the ranking engine.

## Design Pattern: Adapter (Structural)

> "Adapter is a structural design pattern that allows objects with incompatible interfaces to
> collaborate."
> — refactoring.guru

The 9 domain types (`StoreCarousel`, `ItemCarousel`, `DealCarousel`, etc.) have incompatible
interfaces — each has its own `predictionScore`, `sortOrder`, `id` field names, and structure.
`FeedRow` is the target interface. Each `*Row` class is the adapter.

```
StoreCarousel   ──┐
ItemCarousel    ──┤  Adapter  →  FeedRow  →  FeedRowRanker
DealCarousel    ──┤
...             ──┘
```

The engine only ever sees `FeedRow`. Adapters are the only code aware of both sides.

## Is-a / Has-a Analysis

```
FeedRow          IS A rankable feed unit          → interface
RowType          IS A type category               → enum (value object, compared by identity)
StoreCarouselRow IS A FeedRow                     → implements interface
StoreCarouselRow HAS A StoreCarousel              → composition (wraps, not extends)
```

**Why composition over inheritance for adapters?**
`StoreCarousel` is a domain model owned by the pipeline. Extending it would mix ranking
concerns into a domain object that has no knowledge of ranking. Composition keeps the two
concerns separate — SRP. The adapter is disposable when the domain model changes.

**Why interface over abstract class for `FeedRow`?**
Abstract class implies shared implementation. There is none — each adapter has completely
different internal access patterns. An interface is the right choice when the contract is
behavioral only, with no shared state or logic.

## Interface Definition

```kotlin
// libraries/common/.../ubp/vertical/FeedRow.kt
interface FeedRow {
    val id: String
    val type: RowType
    var score: Double              // mutable — steps write scores directly
    val metadata: MutableMap<String, Any>   // side-channel for inter-step communication
    fun recordTrace(stepId: String, scoreBefore: Double, scoreAfter: Double)
    fun applyBackTo()              // writes final score back to the wrapped domain object
}
```

`metadata` is intentionally untyped. Steps that need to pass information to later steps
(e.g., the calibration multiplier used by MODEL_SCORING, for DIVERSITY_RERANK to reference)
write to `metadata`. This avoids coupling steps to each other's types.

## RowType Enum

```kotlin
// libraries/common/.../ubp/vertical/RowType.kt
enum class RowType {
    STORE_CAROUSEL,
    ITEM_CAROUSEL,
    DEAL_CAROUSEL,
    STORE_COLLECTION,
    COLLECTION_V2,
    ITEM_COLLECTION,
    MAP_CAROUSEL,
    REELS_CAROUSEL,
    STORE_ENTITY,
}
```

`FIXED_PINNING` and `DIVERSITY_RERANK` steps use `RowType` to apply type-specific logic
without casting the `FeedRow` itself.

**Scope notes:**
- No `AD_CAROUSEL` — ads are post-POC scope.
- No `NV_CAROUSEL` — NV stores appear within `StoreCarousel` and `ItemCarousel` types; there is no distinct NV domain type. NV-specific step behavior (e.g. NV-only boost) requires carousel ID matching in params or a partner-owned step.

## The 9 Adapters

Each adapter follows the same structure. Shown once in full, then abbreviated for the rest.

### StoreCarouselRow (full example)

```kotlin
// libraries/common/.../ubp/vertical/adapters/StoreCarouselRow.kt
class StoreCarouselRow(
    private val carousel: StoreCarousel,
) : FeedRow {
    override val id: String = carousel.id
    override val type: RowType = RowType.STORE_CAROUSEL
    override var score: Double = carousel.predictionScore ?: 0.0
    override val metadata: MutableMap<String, Any> = mutableMapOf()

    override fun recordTrace(stepId: String, scoreBefore: Double, scoreAfter: Double) {
        metadata["trace_${stepId}"] = mapOf("before" to scoreBefore, "after" to scoreAfter)
    }

    override fun applyBackTo() {
        carousel.sortOrder = score.toInt()
    }
}

// Extension function — keep adapter creation close to the domain type
fun StoreCarousel.toFeedRow(): FeedRow = StoreCarouselRow(this)
```

### Remaining 8 adapters (abbreviated)

```kotlin
class ItemCarouselRow(private val carousel: ItemCarousel) : FeedRow {
    override val id = carousel.id
    override val type = RowType.ITEM_CAROUSEL
    override var score = carousel.predictionScore ?: 0.0
    override val metadata = mutableMapOf<String, Any>()
    override fun recordTrace(...) { ... }
    override fun applyBackTo() { carousel.sortOrder = score.toInt() }
}
fun ItemCarousel.toFeedRow(): FeedRow = ItemCarouselRow(this)

class DealCarouselRow(private val carousel: DealCarousel) : FeedRow { ... }
fun DealCarousel.toFeedRow(): FeedRow = DealCarouselRow(this)

class CollectionV2Row(private val collection: CollectionV2) : FeedRow { ... }
fun CollectionV2.toFeedRow(): FeedRow = CollectionV2Row(this)

class StoreCollectionRow(private val collection: StoreCollection) : FeedRow { ... }
fun StoreCollection.toFeedRow(): FeedRow = StoreCollectionRow(this)

class ItemCollectionRow(private val collection: ItemCollection) : FeedRow { ... }
fun ItemCollection.toFeedRow(): FeedRow = ItemCollectionRow(this)

class MapCarouselRow(private val carousel: MapCarousel) : FeedRow { ... }
fun MapCarousel.toFeedRow(): FeedRow = MapCarouselRow(this)

class ReelsCarouselRow(private val reelsCarousel: AnnouncementCollection) : FeedRow { ... }
fun AnnouncementCollection.toFeedRow(): FeedRow = ReelsCarouselRow(this)

class StoreEntityRow(private val entity: StoreEntity) : FeedRow { ... }
fun StoreEntity.toFeedRow(): FeedRow = StoreEntityRow(this)
```

## Converting Layout to FeedRows

Helper function that builds the `MutableList<FeedRow>` the engine operates on. Lives in the
post-processor as a private function — it's wiring code, not engine code.

```kotlin
// In DefaultHomePagePostProcessor
private fun toFeedRows(layout: HomePageStoreLayoutOutputElements): MutableList<FeedRow> =
    mutableListOf<FeedRow>().apply {
        addAll(layout.storeCarousels.map { it.toFeedRow() })
        addAll(layout.itemCarousels.map { it.toFeedRow() })
        layout.dealCarousel?.let { add(it.toFeedRow()) }
        addAll(layout.collectionsV2.map { it.toFeedRow() })
        addAll(layout.collections.map { it.toFeedRow() })
        addAll(layout.itemCollections.map { it.toFeedRow() })
        addAll(layout.mapCarousels.map { it.toFeedRow() })
        layout.reelsCarousel?.let { add(it.toFeedRow()) }
        addAll(layout.intermixedStores.map { it.toFeedRow() })
    }
```

## Writing Scores Back

After the engine returns the ranked list, write scores back to domain objects and re-sort:

```kotlin
private fun applyRankedRows(
    ranked: List<FeedRow>,
    layout: HomePageStoreLayoutOutputElements,
): HomePageStoreLayoutOutputElements {
    ranked.forEach { it.applyBackTo() }
    // domain objects now have updated sortOrder — layout builder reads these
    return HomePageStoreLayoutOutputElements.Builder(
        storeCarousels = layout.storeCarousels.sortedBy { it.sortOrder },
        itemCarousels  = layout.itemCarousels.sortedBy  { it.sortOrder },
        // ... etc
    ).build()
}
```

## Files to Create

| File | Purpose |
|---|---|
| `libraries/common/.../ubp/vertical/FeedRow.kt` | Interface |
| `libraries/common/.../ubp/vertical/RowType.kt` | Enum |
| `libraries/common/.../ubp/vertical/adapters/StoreCarouselRow.kt` | Adapter |
| `libraries/common/.../ubp/vertical/adapters/ItemCarouselRow.kt` | Adapter |
| `libraries/common/.../ubp/vertical/adapters/DealCarouselRow.kt` | Adapter |
| `libraries/common/.../ubp/vertical/adapters/CollectionV2Row.kt` | Adapter |
| `libraries/common/.../ubp/vertical/adapters/StoreCollectionRow.kt` | Adapter |
| `libraries/common/.../ubp/vertical/adapters/ItemCollectionRow.kt` | Adapter |
| `libraries/common/.../ubp/vertical/adapters/MapCarouselRow.kt` | Adapter |
| `libraries/common/.../ubp/vertical/adapters/ReelsCarouselRow.kt` | Adapter |
| `libraries/common/.../ubp/vertical/adapters/StoreEntityRow.kt` | Adapter |

## Testing

Each adapter needs two tests:
1. `toFeedRow()` correctly reads `id`, `type`, initial `score` from the domain object
2. `applyBackTo()` correctly writes the mutated `score` back to `sortOrder`

```kotlin
@Test
fun `StoreCarouselRow reads id and initial score`() {
    val carousel = StoreCarousel(id = "c1", predictionScore = 0.75, ...)
    val row = carousel.toFeedRow()
    assertEquals("c1", row.id)
    assertEquals(RowType.STORE_CAROUSEL, row.type)
    assertEquals(0.75, row.score)
}

@Test
fun `StoreCarouselRow applyBackTo writes score to sortOrder`() {
    val carousel = StoreCarousel(id = "c1", predictionScore = 0.75, ...)
    val row = carousel.toFeedRow()
    row.score = 0.9
    row.applyBackTo()
    assertEquals(0, carousel.sortOrder)  // 0.9.toInt() = 0 — note: sortOrder semantics TBD
}
```

## Prerequisites

None. This is pure new code — no existing code modified.

## Done When

- All 9 adapters created and unit tested
- `toFeedRows()` helper creates correct list from a sample `HomePageStoreLayoutOutputElements`
- `applyRankedRows()` correctly writes back to domain objects
