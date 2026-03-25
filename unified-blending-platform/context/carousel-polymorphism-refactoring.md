# Carousel Polymorphism Refactoring Analysis

**Date**: 2025-03-25
**Status**: Research / Design Analysis

## Problem Statement

`DefaultHomePagePostProcessor` (1084 lines) contains spread-out conditional logic that branches on carousel type across multiple methods: `trimCarousels()`, `rankPinnedCarouselsByRankingRules()`, `rankAndMergeContent()`, `reOrderGlobalEntitiesV2()`, and others. Each time a new carousel type is added, multiple `when`/`if` branches across different files must be updated.

---

## Current Architecture Layers

The system already uses **four layers of type abstraction**, but they're incomplete and leak conditionals:

| Layer | Mechanism | Where | What It Does |
|-------|-----------|-------|-------------- |
| 1. Type Enums | `StoreCarouselType`, `CarouselType`, `CarouselPageCarouselType` | Model classes | Identifies carousel variant |
| 2. Sealed Class Dispatch | `RankingCandidate` (9 subtypes) | `Ranking.kt` | Wraps carousels for sort-order reassignment |
| 3. Adapter Pattern | `ScorableEntity` (10 implementations) | `scoring/` package | Uniform ranking interface per carousel |
| 4. Strategy Pattern | `CarouselType` interface (CSFv2) | `csfv2/core/models/` | Per-type fetching, ranking, decorating, serializing |

**Gap**: Layers 2-4 handle *horizontal* (within-carousel) concerns well. But **vertical ranking** (ordering carousels on the page) still lives in procedural code inside `DefaultHomePagePostProcessor` with `when` dispatches on type enums.

---

## Refactoring Patterns from refactoring.guru

### 1. Replace Conditional with Polymorphism
- **Problem**: Conditionals branch on object type/properties
- **Solution**: Subclasses per branch, shared method overridden in each
- **Key benefit**: Open/Closed — add new types without modifying existing ranking code

### 2. Replace Type Code with Subclasses
- **Problem**: Enum/string type codes drive behavior via conditionals
- **Solution**: Subclass per type-code value, push behavior into subclass
- **When to avoid**: If type code changes after creation → use State/Strategy instead

### 3. Replace Type Code with State/Strategy
- **Problem**: Type code affects behavior but subclassing isn't viable (class already has hierarchy, or type changes at runtime)
- **Solution**: Extract type-specific behavior into a state/strategy object composed into the host
- **When**: Carousel types are fixed at creation (Strategy fits); runtime changes (State fits)

---

## Where Conditionals Live Today

### A. `trimCarousels()` — lines 976-1046
```kotlin
when (storeCarousel.storeCarouselType) {
    TALL_LOGO_MERCH -> shouldTrimMxLogoCarousel(...)
    LIST_COMPONENT, STORE_LIST_UNIVERSAL_CAROUSEL -> { minStores >= 2 && shouldTrim(...) }
    else -> shouldTrimStoreAndItemCarousel(...)
}
// Plus separate filter blocks for: dealCarousel, storeCollections, collectionsV2,
// itemCarousels, itemCollections, mapCarousels
```
Each carousel *category* has its own filter line with different logic.

### B. `rankPinnedCarouselsByRankingRules()` — lines 868-935
```kotlin
when (rankingCandidate) {
    is StoreCarousel -> sortedStoreCarousels.add(...)
    is ItemCarousel -> sortedItemCarousels.add(...)
    is DealCarousel -> sortedDealCarousel = ...
    is StoreCollection -> sortedStoreCollections.add(...)
    is CollectionV2 -> sortedCollectionsV2.add(...)
}
```
Dispatches into separate mutable lists by type.

### C. `RankableContent` data class
```kotlin
data class RankableContent(
    val storeCarousels: List<StoreCarousel>,
    val dealCarousel: DealCarousel?,
    val storeCollections: List<Collection>,
    val collectionsV2: List<CollectionV2>,
    val itemCarousels: List<ItemCarousel>,
    val itemCollections: List<ItemCollection>,
    val mapCarousels: List<MapCarousel>,
    val reelsCarousels: List<AnnouncementCollection>,
)
```
Every field is a separate typed list — the *data structure itself* forces type-dispatching everywhere it's consumed.

### D. `RankingBundle.ranked()` — Ranking.kt lines 47-149
Converts typed lists → `RankingCandidate` wrappers → ranks uniformly → unwraps back into typed lists via `when`.

---

## How Polymorphism Would Apply

### Target: A Unified `VerticalCarousel` Interface

```kotlin
interface VerticalCarousel {
    val id: String
    val sortOrder: Int

    /** Can this carousel appear on the page given current context? */
    fun shouldDisplay(context: TrimContext): Boolean

    /** Reassign position after ranking */
    fun withSortOrder(newSortOrder: Int): VerticalCarousel

    /** Convert to ScorableEntity for ranking */
    fun toScorable(): ScorableEntity

    /** Is this carousel pinnable? What are its pinning rules? */
    fun pinningBehavior(): PinningBehavior
}
```

### Subclass per Carousel Category

| Current Type | New Class | `shouldDisplay()` Logic |
|---|---|---|
| `StoreCarousel` (REGULAR) | `VerticalStoreCarousel` | `storeSize >= minStores && !capped` |
| `StoreCarousel` (TALL_LOGO_MERCH) | `VerticalMxLogoCarousel` | `storeSize >= mxLogoMin && logoRankingEnabled` |
| `StoreCarousel` (LIST_COMPONENT) | `VerticalListCarousel` | `storeSize >= 2 && shouldTrimStandard(...)` |
| `DealCarousel` | `VerticalDealCarousel` | `deals.size >= minStores && !capped` |
| `ItemCarousel` | `VerticalItemCarousel` | `shouldTrimStoreAndItemCarousel(...)` |
| `StoreCollection` | `VerticalStoreCollection` | `!capped` |
| `CollectionV2` | `VerticalCollectionV2` | `!capped` |
| `MapCarousel` | `VerticalMapCarousel` | `!capped` |
| `ReelsCarousel` | `VerticalReelsCarousel` | Custom reels logic |

### What Changes

**Before** (procedural):
```kotlin
// trimCarousels — must know about every type
content.storeCarousels.filter { when(it.type) { ... } }
content.dealCarousel?.let { if (deals.size >= min) it else null }
content.itemCarousels.filter { shouldTrim(ScorableEntityItemCarousel(it)) }
// ... 6 more filter expressions
```

**After** (polymorphic):
```kotlin
// trimCarousels — type-agnostic
content.carousels.filter { it.shouldDisplay(trimContext) }
```

**Before** (ranking dispatch):
```kotlin
when (candidate) {
    is StoreCarousel -> storeList.add(candidate.reassign(i).carousel)
    is ItemCarousel -> itemList.add(candidate.reassign(i).carousel)
    is DealCarousel -> dealResult = candidate.reassign(i).carousel
    // ... 5 more branches
}
```

**After** (polymorphic):
```kotlin
rankedCarousels = candidates.mapIndexed { i, c -> c.withSortOrder(i) }
```

### RankableContent Simplification

```kotlin
// Before: 8 typed fields
data class RankableContent(
    val storeCarousels: List<StoreCarousel>,
    val dealCarousel: DealCarousel?,
    // ... 6 more
)

// After: single polymorphic list
data class RankableContent(
    val carousels: List<VerticalCarousel>,
) {
    // Typed accessors for backward compat during migration
    val storeCarousels get() = carousels.filterIsInstance<VerticalStoreCarousel>()
    val dealCarousel get() = carousels.filterIsInstance<VerticalDealCarousel>().singleOrNull()
}
```

---

## Which Pattern Fits Best

| Pattern | Fit | Reasoning |
|---|---|---|
| **Replace Type Code with Subclasses** | **Best fit for StoreCarouselType** | The `when(storeCarouselType)` in `trimCarousels()` is textbook. Each enum value drives different trim logic. Subclass `StoreCarousel` or wrap it. |
| **Replace Conditional with Polymorphism** | **Best fit for RankingCandidate / RankableContent** | The multi-type dispatch in ranking and trimming should become method calls on a shared interface. |
| **Replace Type Code with State/Strategy** | **Best fit if carousel types are composed, not inherited** | Since carousel models are data classes (can't subclass easily in Kotlin), a **Strategy** object composed into each carousel handles type-specific behavior without inheritance. This is the most Kotlin-idiomatic approach. |

### Recommended: Strategy Pattern (Type Code → Strategy)

Carousel models are Kotlin data classes — subclassing them breaks `copy()`, `equals()`, etc. Instead:

```kotlin
interface CarouselVerticalStrategy {
    fun shouldDisplay(carousel: VerticalCarousel, context: TrimContext): Boolean
    fun pinningBehavior(): PinningBehavior
    fun toScorable(carousel: VerticalCarousel): ScorableEntity
}

class MxLogoStrategy : CarouselVerticalStrategy {
    override fun shouldDisplay(carousel: VerticalCarousel, ctx: TrimContext): Boolean {
        val stores = (carousel as StoreCarouselWrapper).storeCarousel.stores.size
        return stores >= CarouselsRuntimeUtil.getMinStoresForMxLogoCarousel()
            && DiscoveryExperimentManager.shouldEnableNVMxLogo(ctx.experimentMap)
    }
}
```

This keeps existing data classes untouched, composes behavior externally, and lets `DefaultHomePagePostProcessor` become type-agnostic.

---

## Migration Path

1. **Define `VerticalCarousel` interface** + `CarouselVerticalStrategy`
2. **Create wrapper implementations** for each carousel model (StoreCarousel → VerticalStoreCarousel, etc.)
3. **Move trim logic** from `trimCarousels()` into each strategy's `shouldDisplay()`
4. **Unify `RankableContent`** to hold `List<VerticalCarousel>` with typed accessors for backward compat
5. **Simplify `RankingCandidate`** — `VerticalCarousel.withSortOrder()` replaces sealed class dispatch
6. **Simplify `rankPinnedCarouselsByRankingRules()`** — no more `when` on type
7. **Delete dead `when` branches** as each type is migrated

Each step is independently deployable. Old code paths can coexist via experiment flags.

---

## Relationship to UBP

This refactoring directly enables the Unified Blending Platform goal: if all carousels implement `VerticalCarousel`, the horizontal ranker output (Scorable → ranked list) plugs directly into vertical ranking without type-specific glue code. The `CarouselVerticalStrategy` can be extended to include blending weights, ad eligibility, and other UBP concerns.
