# The HorizontalElement Abstraction

## Core Insight

Every piece of content on the homepage that appears inside a horizontally-scrollable list
(or vertically-stacked list, same idea) is an **element** within a **horizontal component**.
These are two distinct layers:

```
HorizontalComponent  (the carousel/list container — appears at a vertical position on the page)
  └── HorizontalElement[]  (the entries within it — each appears at a horizontal position)
```

The `CrossVerticalHomePageFeedEvent` proto already models this exactly:
- `facet_*` fields → describe the **HorizontalComponent** (carousel level)
- `horizontal_element_score`, `horizontal_position_in_facet`, `store_id`, `item_id` → describe the **HorizontalElement** (entry level)

What we're missing is making this abstraction EXPLICIT in code.

---

## The Abstract Types

### HorizontalComponent (the container)

```
interface HorizontalComponent {
    val id: String             // carousel UUID
    val type: ComponentType    // STORE_CAROUSEL | ITEM_CAROUSEL | DEAL_CAROUSEL | TILE_CAROUSEL | STORE_LIST
    val name: String?          // display title
    val score: Double?         // carousel-level prediction score
    val scoreMultiplier: Double
    val sortOrder: Int?        // vertical position on page
    val enforceManualSortOrder: Boolean
    val elements: List<HorizontalElement>
}
```

Internal model implementations:
| Internal model | ComponentType | elements source |
|---|---|---|
| `StoreCarousel` | `STORE_CAROUSEL` | `stores: List<StoreEntity>` |
| `ItemCarousel` (flat) | `ITEM_CAROUSEL` | `itemStoreEntities: List<ItemStoreEntity>` |
| `ItemCarousel` (grouped) | `ITEM_CAROUSEL` | `groupedItemCarousels[*].itemStoreEntities` (global idx) |
| `DealCarousel` | `DEAL_CAROUSEL` | `deals: List<Deal>` |
| `StoreList` | `STORE_LIST` | `stores: List<StoreEntity>` (each store = its own vertical position) |
| Tile collection | `TILE_CAROUSEL` | tile entries (no storeId) |

### HorizontalElement (the entry)

```
interface HorizontalElement {
    // Primary identity
    val entityId: String        // storeId, itemId, dealId, tileId — the unique ID for this element type
    val entityType: EntityType  // STORE | ITEM | DEAL | TILE

    // Store reference (present for STORE, ITEM; the store this element belongs to or IS)
    val storeId: Long?
    val storeName: String?
    val businessId: Long?
    val primaryVerticalId: Long?
    val businessVerticalId: Long?

    // Ranking
    val score: Double?           // element-level prediction score
    val rawScore: Double?        // pre-adjustment ML score
    val predictorNames: List<String>?
    val modelIds: List<String>?
    val manualSortOrder: Int?    // pre-ML manual sort order (horizontal_manual_sort_order)

    // Attribution
    val isSponsored: Boolean
}
```

Concrete implementations:

| Concrete type | EntityType | entityId | score |
|---|---|---|---|
| `StoreEntity` | `STORE` | `store.id.toString()` | `store.predictionScore` |
| `ItemStoreEntity` | `ITEM` | `item.itemId` | `item.predictionScore` (composite) |
| `Deal` | `DEAL` | `deal.id` | `deal.predictionScore` (if exists) |
| Tile entry | `TILE` | `tile.id` | null |

---

## Why This Matters for Event Design

The `CrossVerticalHomePageFeedEvent` already has fields for both layers. The question is how
to populate them cleanly.

### Single event-building function

Instead of carousel-type-specific builders, one function handles all types:

```kotlin
fun buildElementEvent(
    partial: CrossVerticalHomePageFeedEvent,    // request context fields pre-filled
    component: HorizontalComponent,             // the carousel
    componentVerticalPosition: Int,             // final vertical position
    element: HorizontalElement,                 // the entry
    elementHorizontalPosition: Int,             // position within carousel
): CrossVerticalHomePageFeedEvent

// No type checking on carousel type. No type checking on element type.
// Just fill the fields from the two abstract interfaces.
```

The caller iterates:
```kotlin
sortedComponents.forEachIndexed { verticalIdx, component ->
    component.elements.forEachIndexed { horizontalIdx, element ->
        // Guard: skip elements with no meaningful entityId
        if (element.storeId == null && element.entityType != EntityType.TILE) return@forEachIndexed
        // For tiles: skip entirely (no store data)
        if (element.entityType == EntityType.TILE) return@forEachIndexed

        buildElementEvent(partial, component, verticalIdx, element, horizontalIdx)
    }
}
```

### Mapping internal models to the abstraction

```kotlin
fun StoreEntity.asHorizontalElement(): HorizontalElement = object : HorizontalElement {
    override val entityId = id.toString()
    override val entityType = EntityType.STORE
    override val storeId = id
    override val storeName = name
    override val businessId = businessId
    override val primaryVerticalId = primaryVerticalIds.firstOrNull()
    override val businessVerticalId = businessVerticalId
    override val score = predictionScore
    override val rawScore = rawPredictionScore
    override val predictorNames = predictorNames?.toList()
    override val modelIds = predictorModelIds?.toList()
    override val manualSortOrder = null  // store manual sort from retrievalData.storeCarouselMapping
    override val isSponsored = isSponsored ?: false
}

fun ItemStoreEntity.asHorizontalElement(): HorizontalElement = object : HorizontalElement {
    override val entityId = itemId
    override val entityType = EntityType.ITEM
    override val storeId = storeId
    override val storeName = storeName
    override val businessId = businessId?.toLongOrNull()
    override val primaryVerticalId = primaryVerticalIds.firstOrNull()
    override val businessVerticalId = businessVerticalId?.toLongOrNull()
    override val score = predictionScore          // composite (organic + inventory)
    override val rawScore = organicPredictionScore
    override val predictorNames = predictorNames?.toList()
    override val modelIds = predictorModelIds?.toList()
    override val manualSortOrder = sortOrder      // directly available, no retrieval lookup
    override val isSponsored = false              // set true for adItemStoreEntities
}
```

### What populates `CrossVerticalHomePageFeedEvent` from each layer

**From HorizontalComponent:**
| Event field | Source |
|---|---|
| `facet_id` | `"${componentTypePrefix(component.type)}:${component.id}"` |
| `facet_type` | `component.type.dmType` (e.g. `"store_carousel"`) |
| `facet_name` | `component.name` |
| `facet_vertical_position` | `componentVerticalPosition` |
| `facet_score` | `component.score` |
| `facet_score_multiplier` | `component.scoreMultiplier` |
| `vertical_manual_sort_order` | from `retrievalData.carouselMetadata` |

**From HorizontalElement:**
| Event field | Source |
|---|---|
| `store_id` | `element.storeId` |
| `store_name` | `element.storeName` |
| `business_id` | `element.businessId` |
| `store_primary_vertical_id` | `element.primaryVerticalId` |
| `item_id` | `element.entityId` (when `entityType == ITEM`) |
| `horizontal_position_in_facet` | `elementHorizontalPosition` |
| `horizontal_element_score` | `element.score` |
| `raw_horizontal_element_score` | `element.rawScore` |
| `predictor_names` | `element.predictorNames` |
| `model_ids` | `element.modelIds` |
| `horizontal_manual_sort_order` | `element.manualSortOrder ?: DEFAULT_MANUAL_SORT_ORDER_TO_TRACK` |
| `is_sponsored` | `element.isSponsored` |

---

## On Deals

`DealCarousel` fits this model:
- `DealCarousel` → `HorizontalComponent` with `ComponentType.DEAL_CAROUSEL`
- Each `Deal` → `HorizontalElement` with `EntityType.DEAL`

A `Deal` has `storeId`, `businessId`, and deal-specific fields (`dealId`, `dealName`, `offerType`).

Current status: `CrossVerticalHomePageFeedEvent` has `store_id` and `item_id` fields but not
`deal_id`. If deals need their own logging, either:
1. Reuse `item_id` for `deal_id` (with `entity_type = "deal"` for disambiguation)
2. Add a `deal_id` field to the proto

For now: `DealCarousel` is SKIPPED (as aligned with processFilteredSections which only emits
if `storeId > 0` from the deal's logging, which is typically absent). The abstraction is
designed so adding deal support later only requires mapping `Deal → HorizontalElement`.

---

## Relation to `processFilteredSections` filter logic

The abstraction makes the filter rules clearer:

```
EMIT if:
  element.storeId != null && element.storeId > 0    (STORE, ITEM elements with a store)

SKIP:
  element.entityType == TILE                         (no store, navigation only)
  component.type == DEAL_CAROUSEL                    (deal grain, not store grain — for now)
  component == bannerCarousel, etc.                  (EXCLUDED_SECTION_IDS equivalent)
```

This replaces 15+ type-specific generator classes with one conditional.
