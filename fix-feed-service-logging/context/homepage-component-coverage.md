# Homepage Component Coverage Analysis
# (Cross-referenced against processFilteredSections() filtering logic)

## Excluded sections (EXCLUDED_SECTION_IDS — same exclusions apply to our approach)

| Section ID                                        | Internal model field        | Reason                                                      |
| ------------------------------------------------- | --------------------------- | ----------------------------------------------------------- |
| `BANNER_FACET_SECTION_ID`                         | `bannerCarousel`            | Pure UI                                                     |
| `NAVIGATION_TILES_FACET_SECTION_ID`               | `rankedVerticalEntryPoints` | Already logged via `generateVerticalEntryPointLoadEvents()` |
| `STORE_LIST_HEADER_FACET_SECTION_ID`              | (UI header, no field)       | Pure UI header                                              |
| `CAROUSEL_OFFERS_CUISINE_FILTER_FACET_SECTION_ID` | `cuisineFiltersV2`          | Filter chips, no stores                                     |

---

## Components and how processFilteredSections() handles them

### EMIT per-store events ✅

| Internal model | Facet ID prefix | Generator | Event grain | Guard |
|---|---|---|---|---|
| `storeCarousels` | `carousel.standard:store_carousel*` | `StoreCarouselGenerator` | per store in carousel | `storeId > 0` |
| `storeCarousels` (UC format) | `carousel.universal:store_carousel*` | `UniversalStoreCarouselGenerator` | per store in carousel | `storeId > 0` |
| `storeList.stores` | `row.store*` or `card.store:store*` (CC format) | `RowStoreGenerator` | per store row | no explicit guard, storeId from logging |
| `storeList.stores` (retail variant) | `carousel.retail_store_list*` | `RetailStoreListCarouselGenerator` | per store card | no explicit guard |
| `collectionsV2` children that are store carousels | `collection.standard:store_collection*` or `carousel.standard:store_collection*` → child `carousel.standard:store_carousel*` | `StoreCarouselGenerator` (on child facet) | per store in child carousel | `storeId > 0` |
| `feedPlacements` (regular) | `FeedPlacement.MERCHANDISINGUNIT_COMPONENT_PREFIX` | `FeedPlacementGenerator` (base impl) | per child in facet | `storeId > 0` |
| `feedPlacements` (UC store type) | `FeedPlacement.UC_MERCHANDISINGUNIT_COMPONENT_STORE_PREFIX` | `UniversalStoreFeedPlacementGenerator` | per store in UC | `storeId > 0` |
| `feedPlacements` (standalone unit = single store spotlight) | `FeedPlacement.STANDALONE_UNIT_CONTAINER` | `FeedPlacementStandaloneUnitGenerator` | single store (position 0) | `storeId > 0` |
| `reelsCarousel` | `carousel.standard:reels:reels_carousel*` | `ReelsCarouselGenerator` | per announcement, only if store decorated | `storeId > 0` |

---

### EMIT item-level events (per item, storeId as context) ⚠️

All three generators below use a `itemId.isNotEmpty()` guard and set `horizontal_position_in_facet` to the **item index**, not the store index. `store_id` is logged as context but the event grain is the item. We follow the same per-item model using `ItemCarousel.itemStoreEntities`.

| Internal model | Facet ID prefix | Generator | Event grain | Guard |
|---|---|---|---|---|
| `itemCarousels` (legacy format) | `carousel.standard:item_carousel*` | `ItemCarouselGenerator` | per item | `itemId.isNotEmpty()` |
| `itemCarousels` (UC format) | `cx.common.universal_carousel:item_carousel*` | `UniversalItemCarouselGenerator` | per item | `itemId.isNotEmpty()` |
| `itemCarousels` (multi-MX deals / grouped) | `carousel.standard:deals_v2_carousel` | `CarouselStandardGenerator` | per item | `itemId.isNotEmpty()` |

#### Internal model field mapping for item-level events

`ItemCarousel.itemStoreEntities: List<ItemStoreEntity>` is the direct equivalent of iterating `facet.childrenList` in `ItemCarouselGenerator`. Each `ItemStoreEntity` has both item-level and store-level data — no lookup needed.

| `CrossVerticalHomePageFeedEvent` field | `ItemStoreEntity` source |
|---|---|
| `item_id` | `itemStoreEntity.itemId` |
| `item_name` | `itemStoreEntity.itemName` |
| `item_price` | `itemStoreEntity.itemPrice` (cents) |
| `item_image_url` | `itemStoreEntity.itemImageUrl` |
| `horizontal_element_score` | `itemStoreEntity.predictionScore` (composite: organic + inventory) |
| `raw_horizontal_element_score` | `itemStoreEntity.organicPredictionScore` (raw ML score from item ranker) |
| `horizontal_position_in_facet` | index in `itemStoreEntities` list (rendered order after ranking) |
| `horizontal_manual_sort_order` | `itemStoreEntity.sortOrder ?: DEFAULT_MANUAL_SORT_ORDER_TO_TRACK` |
| `store_id` | `itemStoreEntity.storeId` |
| `store_name` | `itemStoreEntity.storeName` |
| `business_id` | `itemStoreEntity.businessId?.toLongOrNull()` |
| `store_primary_vertical_id` | `itemStoreEntity.primaryVerticalIds.firstOrNull()` |
| `store_business_vertical_id` | `itemStoreEntity.businessVerticalId?.toLongOrNull()` |
| `store_price_range` | (not on ItemStoreEntity — omit or set 0) |
| `store_star_rating` | `itemStoreEntity.averageRating` |
| `store_num_ratings` | `itemStoreEntity.numberOfRatings` |
| `store_is_dashpass_partner` | `itemStoreEntity.isDashpassPartner` |
| `store_header_image_url` | `itemStoreEntity.coverSquareImgUrl` |
| `store_delivery_fee_amount` | `itemStoreEntity.deliveryFeeAmount` |
| `store_asap_available` | `itemStoreEntity.status?.asapAvailable` |
| `store_asap_minutes` | `itemStoreEntity.status?.asapMinutes` |
| `store_distance_in_miles` | `itemStoreEntity.distanceFromConsumerInMiles` |
| `predictor_names` | `itemStoreEntity.predictorNames?.toList()` |
| `model_ids` | `itemStoreEntity.predictorModelIds?.toList()` |
| `cs_st_cuisine_affinity_similarity` | (not on ItemStoreEntity — omit or 0.0) |
| `percentage_match` | (not on ItemStoreEntity — omit or 0.0) |
| `is_sponsored` | set `true` for items from `adItemStoreEntities` (see below) |

#### Sponsored items (`adItemStoreEntities`)

`ItemCarousel.adItemStoreEntities: List<ItemStoreEntity>?` are sponsored/ad items mixed into the carousel. In the FacetSection path they appear in `facet.childrenList` alongside organic items; the renderer interleaves them. In the internal model path:
- Emit events for `itemStoreEntities` (organic) with `is_sponsored = false`
- Emit events for `adItemStoreEntities` (sponsored) with `is_sponsored = true`
- `horizontal_position_in_facet` for sponsored items: use the item's actual rendered position if available (`itemStoreEntity.itemIndex`), otherwise append after organic items

#### Grouped item carousels (`groupedItemCarousels`)

`ItemCarousel.groupedItemCarousels: List<ItemCarousel>` is used for multi-MX deals / category-grouped structures (maps to `CarouselStandardGenerator`). The parent `itemStoreEntities` is typically empty in this case; items live in each group's `itemStoreEntities`.

Handling:
```
if (carousel.groupedItemCarousels.isNotEmpty()) {
    // iterate groups, emit from each group's itemStoreEntities
    // horizontal_position_in_facet counts globally across groups (same as CarouselStandardGenerator.globalChildIdx)
    var globalIdx = 0
    carousel.groupedItemCarousels.forEach { group ->
        group.itemStoreEntities.forEach { item ->
            emitItemEvent(item, globalIdx++, ...)
        }
    }
} else {
    carousel.itemStoreEntities.forEachIndexed { idx, item ->
        emitItemEvent(item, idx, ...)
    }
}
```

---

### EMIT tile/collection events — NO storeId (effectively noise for store-load table) ⚠️

| Internal model                                                 | Facet ID prefix                                                              | Generator                        | Logged fields                                            | storeId                                                |
| -------------------------------------------------------------- | ---------------------------------------------------------------------------- | -------------------------------- | -------------------------------------------------------- | ------------------------------------------------------ |
| `collectionsV2` children that are tile-based store collections | `card.tile:store_carousel` (children of `carousel.standard:store_carousel*`) | `StoreCollectionGenerator`       | `TILE_ID`, `TILE_NAME`, `TILE_STORE_IDS` (no `STORE_ID`) | 0 → **filtered out by** `storeId > 0` guard            |
| Tile carousels (collections rendered as tiles)                 | `custom_tile_collection*` or `carousel.standard:tile_collection*`            | `TileCollectionGenerator`        | `TILE_ID`, `TILE_NAME`, `TILE_STORE_IDS`                 | 0 → **NOT guarded** — events DO emit but `storeId = 0` |
| UC tile carousels                                              | `cx.common.universal_carousel:tile_carousel*`                                | `UniversalTileCarouselGenerator` | `TILE_ID`, `TILE_NAME`, `TILE_STORE_IDS`                 | 0 → **NOT guarded** — events DO emit but `storeId = 0` |

**Implication for our approach:**
`TileCollectionGenerator` and `UniversalTileCarouselGenerator` emit events with `storeId = 0`. These represent category navigation tiles (e.g., "Pizza", "Grocery"), not individual stores. **SKIP these in our new approach** — they produce noise in a store-load table. This matches the practical intent of `StoreCollectionGenerator` (filtered out by storeId > 0 guard), just explicitly applied to tile carousels too.

In the internal model: `collectionsV2` children that are tile collections don't have `StoreEntity` — they're category or cuisine tiles with a `tileId` + set of storeIds. Skip them.

---

### DEAL CAROUSEL — conditional ⚠️

| Internal model | Facet ID prefix | Generator | Notes |
|---|---|---|---|
| `dealCarousel` | `carousel.standard:deal_carousel*` (in `DEAL_CAROUSEL_FACET_SECTION_ID`) | `DealCarouselGenerator` (base impl) | Uses base `generateEvents()` → `storeId > 0` guard. `Deal` objects in FacetV2 children MAY contain `storeId` in their logging. If they do, events are emitted; otherwise filtered. |

**Key finding:** `DealCarousel.deals: List<Deal>` in the internal model contains `Deal` objects with `storeName`, `storeId`, `businessId` etc. So individual deals DO have a storeId reference. The `DealCarouselGenerator` tries to read `storeId` from `childLogging` and emits if `> 0`.

**Implication for our approach:**
We CAN emit per-deal events from `dealCarousel.deals` using `deal.storeId`. However, the event grain here is per-DEAL not per-STORE (a store may appear in multiple deals). For a store-load table, deals should either be:
- Emitted as separate deal events (not in the store-load table), OR
- Included in the store-load table as `facet_type = "deal_carousel"` with one row per deal

**Recommendation: SKIP dealCarousel for the store-load table.** Deal logging is better served by a separate event/table. This matches the practical outcome of `DealCarouselGenerator` (often storeId = 0 if deal logging doesn't include it).

---

### SILENTLY SKIPPED by processFilteredSections() — no matching case

| Internal model | Reason skipped | Has StoreEntity? |
|---|---|---|
| `dineOutCarousel` | No matching facet ID prefix | NO — `RewardItem` objects with `storeName` string only |
| `dashPartyCarousel` | No matching facet ID prefix | Likely NO — social feature |
| `mapCarousels` | No matching facet ID prefix | NO — UI map component |
| `giftCardCarousels` | No matching facet ID prefix | NO — gift card items |
| `avProductCarousel` | No matching facet ID prefix | TBD — AV product, likely no StoreEntity |
| `smartSuggestionCollection` | No matching facet ID prefix | TBD |
| `serviceFeeBanner` | No matching facet ID prefix | NO — pure UI |
| `spotlightBanner` | No matching facet ID prefix | NO — pure UI |
| `doubleDashReminder` | No matching facet ID prefix | NO — pure UI |

**These are silently skipped today and should be silently skipped in our approach too.**

---

## Definitive component list for HomepageStoreLoadEventUtil

### EMIT per-store events (one CrossVerticalHomePageFeedEvent per store):
```
1. storeCarousels[*].stores[idx]
   - facet_type = "store_carousel"
   - horizontal_position_in_facet = idx
   - vertical position from sortOrder / sortedPlacements index

2. storeList.stores[idx]
   - facet_type = "store_list_element" (STORE_LIST_ELEMENT_DM_TYPE)
   - horizontal_position_in_facet = 0 (each row is its own vertical position)
   - vertical position = storeList.sortOrder + idx (each row is separate facet in FacetSection)

   NOTE: RowStoreGenerator treats EACH STORE ROW as a separate facet with its own
   vertical position. So storeList is NOT one facet with N children — it's N separate
   facets each at a different vertical position. We must replicate this.

3. itemCarousels[*].stores[idx]
   - facet_type = DATA_MODEL__ITEM_CAROUSEL
   - horizontal_position_in_facet = idx (store index, not item index — diverging from existing per-item behavior)
   - vertical position from carousel.sortOrder / sortedPlacements index
   - Include item-level fields where available (first item from that store, or omit)

4. collectionsV2[*] children that are StoreCarousel → recurse to case 1
   - facet_section_category_id = collection's primary vertical (if category section)

5. feedPlacements[*] where store != null
   - facet_type = FeedPlacement.MERCHANDISINGUNIT_COMPONENT_PREFIX
   - horizontal_position_in_facet = 0 (single store)
   - vertical position from placement.sortOrder / sortedPlacements index

6. intermixedStores[idx] (each store treated as its own row, like RowStoreGenerator)
   - facet_type = "store_list_element"
   - horizontal_position_in_facet = 0
   - vertical position = store.sortOrder

7. reelsCarousel.decoratedStoreEntities[idx] (only if non-empty)
   - facet_type = "reels_carousel"
   - horizontal_position_in_facet = idx
   - vertical position from reelsCarousel.sortOrder
```

### SKIP (align with processFilteredSections exclusions + silently-skipped components):
```
- bannerCarousel (excluded section)
- rankedVerticalEntryPoints (excluded section, logged separately)
- cuisineFiltersV2 (excluded section)
- TileCollections / collectionsV2 children that are tile-based (storeId = 0)
- dealCarousel (items are deal-grain, not store-grain; separate event if needed)
- dineOutCarousel (no StoreEntity, silently skipped)
- dashPartyCarousel (silently skipped)
- mapCarousels (silently skipped)
- giftCardCarousels (silently skipped)
- avProductCarousel (silently skipped)
- smartSuggestionCollection (silently skipped)
- serviceFeeBanner, spotlightBanner, doubleDashReminder (silently skipped)
```

---

## StoreList vertical position handling (important detail)

In the serialized FacetSection path, `RowStoreGenerator` treats **each store as a separate FacetV2 at a unique vertical position**. Each store row in the store_feed section is a top-level facet, not a child. So for N stores in storeList, `processFilteredSections` emits N events, each with a different `facet_vertical_position` (counting upward from where the store_feed section starts).

In the internal model approach:
- `storeList.sortOrder` = the vertical slot for the store list as a whole
- Individual store rows within it are sub-positions: `storeList.sortOrder + storeIdx`
- `facet_type = "store_list_element"` (matching `RowStoreGenerator`)
- `facet_id = "row.store:store:{storeId}"` (matching the serialized facet ID pattern)

---

## horizontal_manual_sort_order source

`StoreCarouselGenerator` reads manual sort order from `retrievalData.storeCarouselMapping`:
```kotlin
retrievalData.storeCarouselMapping
    .find { StoreCarouselDataAdapter.generateFacetIdFromCarouselId(it.carouselId) == facetId }
    ?.storeWithSortOrder?.find { it.storeId == ... }
    ?.sortOrder ?: -1
```
This is available in `HomePageStoreDiscoveryResponse.storeCarouselMapping`. Must be passed through.
Similarly for item carousels: `retrievalData.itemCarouselMapping` for `horizontal_manual_sort_order`.

---

## vertical_manual_sort_order and vertical_overridden_sort_order

`additionalContainerLogging` sets these for `CrossVerticalHomePageFeedEvent`:
- `vertical_manual_sort_order`: from `retrievalData.carouselMetadata` → carousels where `enforceManualSortOrder = true`
- `vertical_overridden_sort_order`: from `layoutData.storeCarousels / itemCarousels / collections / collectionsV2 / reelsCarousel` → carousels where `enforceManualSortOrder = true`

These can be populated in our new approach from the internal model directly.
