# Homepage Component Coverage Analysis

## All Components in HomePageStoreLayoutOutputElements

### Store-Containing (can emit CrossVerticalHomePageFeedEvent per store)

| Component | Field | Has stores | sortOrder source | Notes |
|---|---|---|---|---|
| StoreCarousel | `storeCarousels` | `stores: List<StoreEntity>` | `carousel.sortOrder` | `adStoreCarousels` are merged in by post-processor via `maybeAddSponsoredCarousel` |
| StoreList | `storeList` | `stores: List<StoreEntity>` | `storeList.sortOrder` (always last, set by post-processor) | V2: `sortedPlacements.size`; V1: `max(all other sortOrders) + 1` |
| ItemCarousel | `itemCarousels` | `stores: List<StoreEntity>` | `carousel.sortOrder` | Stores are the merchants backing items; emit one event per store |
| CollectionV2 | `collectionsV2` | children: `List<BaseCarousel>` (may contain StoreCarousel, ItemCarousel) | `collection.sortOrder` for the wrapper, child `sortOrder` for child carousels | Recurse into children |
| Collection | `collections` | children (similar) | `collection.sortOrder` | |
| ItemCollection | `itemCollections` | `stores: List<StoreEntity>` | `collection.sortOrder` | |
| FeedPlacement | `feedPlacements` | `store: StoreEntity?` or `collection: BaseCarousel?` | `placement.sortOrder` | Check `placement.store != null`; also check `placement.collection` for store-containing carousels |
| CategorySection children | `categorySections` | each section contains `storeCarousels`, `collections`, etc. | child carousel's `sortOrder` | `CategorySection.verticalId` = the category section ID for `facet_section_category_id` |
| intermixedStores | `intermixedStores` | `List<StoreEntity>` directly | `store.sortOrder` | Individual stores injected between carousels |
| AnnouncementCollection | `reelsCarousel` | `decoratedStoreEntities: List<StoreEntity>?` (optional) | `reelsCarousel.sortOrder` | Only emit if `decoratedStoreEntities` is non-empty |

### Non-Store Components (skip for per-store events)

| Component | Reason |
|---|---|
| `bannerCarousel: BannerCarousel?` | Pure UI, no stores |
| `dealCarousel: DealCarousel?` | `deals: List<Deal>` - deal entities, not StoreEntity; logged separately if needed |
| `giftCardCarousels: List<GiftCardCarousel>?` | Gift card items, not stores |
| `rankedVerticalEntryPoints: List<VerticalWithConfig>` | Navigation tiles; already covered by `generateVerticalEntryPointLoadEvents` |
| `mapCarousels: List<MapCarousel>` | UI map component |
| `dineOutCarousel: DineOutCarousel?` | Dine-out reservations, not delivery stores |
| `dashPartyCarousel: DashPartyCarousel?` | Social feature, likely no StoreEntity |
| `spotlightBanner: SpotlightBanner?` | Pure UI |
| `serviceFeeBanner: ServiceFeeBanner?` | Pure UI |
| `doubleDashReminder: DoubleDashReminder?` | Pure UI |
| `avProductCarousel: AvProductCarousel?` | Autonomous vehicle, check if stores |
| `smartSuggestionCollection` | Smart suggestions, likely not StoreEntity |
| `cuisines / cuisineFiltersV2` | Filter chips |
| `discoveryPageComponentConfigs` | Config only |

---

## Correct Vertical Position: sortedPlacements vs sortOrder

The V2 post-processor (`reOrderGlobalEntitiesV2`) populates `HomePageStoreLayoutOutputElements.sortedPlacements`
which is a `List<SortablePlacement>` — the final, authoritatively ordered list of ALL content.

**V2 path (sortedPlacements is non-empty):**
```
sortedPlacements[0] → facet_vertical_position = 0
sortedPlacements[1] → facet_vertical_position = 1
...
```
Each entry is a `StoreCarousel`, `ItemCarousel`, `FeedPlacement`, `BannerCarousel`, etc.
Walk this list and emit events for store-bearing types.

**V1 path (sortedPlacements is empty — deprecated but still active):**
Fall back to each component's `.sortOrder` field.
`storeList.sortOrder` = `max(all other sortOrders) + 1` (always last).

**Recommendation:** Check `sortedPlacements.isNotEmpty()` first, use it if available, else fall back to sortOrder.

---

## CategorySection handling

CategorySections appear on the cross-vertical homepage where verticals (Food, Grocery, etc.)
each have their own nested carousels. The `categorySections` field is independent of `sortedPlacements`
(sections are a presentation-layer grouping).

For logging: each carousel inside a CategorySection has its own `sortOrder`. The `CategorySection.verticalId`
maps directly to `facet_section_category_id` in CrossVerticalHomePageFeedEvent.

---

## Total store-bearing component types to handle

```
HomepageStoreLoadEventUtil.generateEvents() needs to iterate:

1. sortedPlacements (V2) OR (storeCarousels + itemCarousels + collectionsV2 + ... sorted by sortOrder) (V1)
   - StoreCarousel → stores[idx] → horizontal_position_in_facet = idx
   - ItemCarousel → stores[idx] → horizontal_position_in_facet = idx
   - CollectionV2 → recurse into children
   - FeedPlacement → store (single, horizontal_position = 0) or collection.stores
   - StoreList → stores[idx] → horizontal_position_in_facet = idx

2. categorySections (if present) — for each section's child storeCarousels, etc.
   - facet_section_category_id = section.verticalId

3. intermixedStores (if present) — each store has its own sortOrder

4. reelsCarousel.decoratedStoreEntities (if non-null and non-empty)
```
