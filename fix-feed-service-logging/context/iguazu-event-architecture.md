# Iguazu Event Architecture Analysis

## Two Existing Event Paths

### 1. Legacy Homepage → `cx_home_page_feed` (HomePageFeedEvent)
- **Generator**: `LegacyHomePageIguazuEventUtil.generateHomePageFeedSegmentEvents()`
- **Input**: `ExploreContext` + `HomePageStoreLayoutOutputElements`
- **Emit point**: `DefaultHomePageResponseSerializer.recordFallbackAndEvents()`
- **Data source**: Iterates directly over internal Kotlin models (`StoreCarousel`, `StoreEntity`)
- **Position source**: `storeIndex` (horizontal), `carousel.sortOrder` (vertical, implicit via sortOrder field)
- **Status**: Works but uses older proto fields (1-55), missing request_id, vertical IDs, predictor info

### 2. Realtime / Cross-Vertical Homepage → `cx_cross_vertical_homepage_feed` (CrossVerticalHomePageFeedEvent)
- **Generator**: `IguazuEventUtil.generateXVerticalHomePageFeedSegmentEvents()`
- **Inputs**: `ExploreContext` + `List<FacetSection>` (serialized gRPC response) + `HomePageStoreDiscoveryResponse` + `HomePageStoreLayoutOutputElements`
- **Emit point**: `SerializerHelperUtil.sendHomePageFeedEventLogging()` → called from `InstrumentedRealTimeResponseSerializer`
- **Data source (position)**: Manually counting `verticalIdxOffset + verticalIdx` by parsing FacetSection structure
- **Data source (store attributes)**: Looking back up into `layoutData` by storeId after parsing FacetSection
- **Status**: Architecturally fragile — builds from serialized response then looks up internal model

## The Core Problem with the FacetSection Approach

`processFilteredSections()` in `IguazuEventUtil`:
1. Filters out EXCLUDED sections (banners, nav tiles, etc.)
2. Manually maintains `verticalIdxOffset` counter as it walks sections
3. Dispatches to 15+ `ContainerEventsGenerator` subclasses based on facet ID prefix matching
4. Inside generators: parses store data from FacetV2.logging fields (string key-value map)
5. For ML/ranking data that isn't in FacetV2: looks back up in `layoutData` by storeId

### Known Discrepancy (from code comment):
```kotlin
// TODO: use logged vertical position for all cases after we've resolved
// discrepancies between client side vs server side vertical position
val verticalPosition = if (fromRealTimePipeline && loggedVerticalPosition != null) {
    loggedVerticalPosition
} else {
    verticalIdxOffset + verticalIdx   // manually counted — breaks when section types change
}
```

The correct vertical position is already set on `StoreCarousel.sortOrder` by the post-processor.
No need to re-derive it.

## Better Approach: Emit Directly from Internal Models

All fields for `CrossVerticalHomePageFeedEvent` are available in `HomePageStoreLayoutOutputElements`:

| Field | FacetSection path | Internal model path |
|---|---|---|
| `facet_vertical_position` | manually counted offset | `storeCarousel.sortOrder` |
| `horizontal_position_in_facet` | loop index over FacetV2 children | loop index over `carousel.stores` |
| `facet_id` | FacetSection.id prefix | `"carousel.standard:store_carousel:${carousel.id}"` |
| `facet_type` | decoded from FacetSection.id prefix | statically known: `"store_carousel"` etc. |
| `facet_name` | parsed from FacetV2.logging map | `carousel.title` |
| `facet_score` | parsed from FacetV2.logging map | `carousel.predictionScore` |
| `facet_score_multiplier` | parsed from FacetV2.logging map | `carousel.scoreMultiplier` |
| `store_id`, etc. | looked up in layoutData by storeId | direct from `StoreEntity` |
| `horizontal_element_score` | parsed from FacetV2 child logging | `store.predictionScore` |
| `store_primary_vertical_id` | parsed from FacetV2 child logging | `store.primaryVerticalId` |
| `predictor_names`, `model_ids` | parsed from FacetV2 child logging | `store.rankingFeatures.*` |
| `is_sponsored` | parsed from FacetV2 child logging | `store.isSponsored` |

### Fields only available post-serialization (acceptable to drop/approximate):
- `facet_section_id` — the section grouping (e.g., `"global_carousels:favorites"`) — presentation concern
- `facet_section_category_id` — can approximate from `carousel.primaryVerticalId`

## Proposed: New HomePageStoreLoadIguazuEventUtil

New utility that takes only internal models:
```
generateEvents(
    context: ExploreContext,
    elements: HomePageStoreLayoutOutputElements,
    retrievalData: HomePageStoreDiscoveryResponse,  // for consumer preference fields
): List<Event<Message>>
```

Iterates:
1. `elements.storeCarousels` → `CrossVerticalHomePageFeedEvent` per store, with `sortOrder` as vertical position
2. `elements.storeList.stores` → `CrossVerticalHomePageFeedEvent` per store, with `StoreList` sort order
3. `elements.dealCarousel?.deals` → events per deal store
4. `elements.collectionsV2` → events per child carousel store

Called from `DefaultHomePageResponseSerializer.recordFallbackAndEvents()` in the existing fire-and-forget coroutine, replacing the `LegacyHomePageIguazuEventUtil` call.

## What to NOT Change

- `IguazuEventUtil.generateXVerticalHomePageFeedSegmentEvents()` — leave as-is for realtime pipeline
  which produces `GetFacetFeedResponseV2` (FacetSection-based), where the serialized response
  IS the source of truth for layout structure
- `ContainerEventsGenerator` — leave as-is for realtime pipeline usage
- `IguazuModule` — no changes needed

## Emit Point

`DefaultHomePageResponseSerializer.recordFallbackAndEvents()` — already fire-and-forget, already has
access to both `homepageContext: ExploreContext` and `data: HomePageStoreLayoutOutputElements`.
Need to also pass `retrievalData` (currently not threaded to this point for legacy pipeline).
Check if `HomepageResponseSerializerInput` already carries it or if it needs to be added.
