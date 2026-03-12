# Pipeline Fork: Emit Events at Post-Processing

## Goal

Emit Iguazu events as soon as stores have their final vertical+horizontal rankings —
i.e., the moment `postProcessingJob` completes — without waiting for serialization.
Serialization and event emission proceed independently in parallel.

## Current Pipeline DAG (simplified)

```
contextJob
    └── contentRetrievalJob
            └── contextDecorationJob
                    ├── rankingDecorationJob
                    ├── groupingJob
                    │       ├── storeRankingJob ──────┐
                    │       └── campaignRankingJob ───┤
                    │                                  └── experienceDecorationJob
                    │                                          └── layoutProcessingJob
                    │                                                  └── postProcessingJob
                    │                                                          └── outputSerializerJob ← critical path
```

## Proposed Change: Fork at postProcessingJob

```
postProcessingJob ──┬── outputSerializerJob    (critical path → gRPC response)
                    └── iguazuEventEmissionJob  (failOnError=false, fire-and-forget)
```

Both `outputSerializerJob` and `iguazuEventEmissionJob` depend only on `postProcessingJob`.
They run independently. The gRPC response is not blocked by event emission.

## Implementation in DefaultHomePagePipeline

Add a new job to `constructWorkflow()`:

```kotlin
val iguazuEventEmissionJob = suspend {
    val homepageFeedContext = workflow.getResult(homepageFeedContextDecorationJob)
    val outputElements = workflow.getResult(postProcessingJob)
    val retrievalData = workflow.getResult(restaurantStoreContentRetrievalJob)  // already completed at this point

    // Fire-and-forget: does not block workflow job completion
    CoroutineScope(coroutinesThreadPool.appCoroutineContext()).launch {
        iguazuModule.sendEvent(XVERTICAL_HOMEPAGE_EVENT_NAME) {
            HomepageStoreLoadEventUtil.generateEvents(
                context = homepageFeedContext,
                elements = outputElements,
                retrievalData = retrievalData,
            )
        }
    }
    KSuccess(Unit)
}

workflow.addJob(
    iguazuEventEmissionJob,
    JOB_IGUAZU_EVENT_EMISSION,
    failOnError = false,  // logging failure must never affect the response
    dependsOn = setOf(postProcessingJob),
)
```

The job itself returns `KSuccess(Unit)` immediately after launching the inner coroutine.
The inner coroutine emits events asynchronously.

## Why not emit from inside postProcessingJob or DefaultHomePagePostProcessor?

**Option: emit from postProcessingJob directly**
- Works, but the post-processing job is on the critical path
- Any delay in setting up the launch would add latency
- Mixing "compute layout" and "start logging" concerns in the same job

**Option: emit from DefaultHomePagePostProcessor.reOrder()**
- Post-processor already has `iguazuModule` injected and already does async launches
- BUT: post-processor shouldn't know about event schemas; `retrievalData` isn't available there
- Already handling enough complexity

**Option: new dedicated workflow job (recommended)**
- Clean separation of concerns
- Explicit in the DAG — anyone reading the pipeline can see that logging is a distinct parallel step
- `failOnError = false` enforced at the framework level
- `retrievalData` is accessible (already resolved)
- Serialization is not blocked at all

## What HomepageStoreLoadEventUtil.generateEvents() needs

Inputs:
- `context: ExploreContext` — consumer, geo, device, session, request context
- `elements: HomePageStoreLayoutOutputElements` — post-processed layout with final sort orders
- `retrievalData: HomePageStoreDiscoveryResponse` — consumer preference signals (preferred stores/categories)

### Iteration strategy

```kotlin
fun generateEvents(
    context: ExploreContext,
    elements: HomePageStoreLayoutOutputElements,
    retrievalData: HomePageStoreDiscoveryResponse,
): List<Event<Message>> {
    val partial = buildRequestPartial(context, retrievalData)
    val events = mutableListOf<Event<Message>>()

    if (elements.sortedPlacements.isNotEmpty()) {
        // V2: use sortedPlacements list — index = authoritative vertical position
        elements.sortedPlacements.forEachIndexed { verticalIdx, placement ->
            emitForPlacement(placement, verticalIdx, null, partial, events)
        }
    } else {
        // V1 fallback (deprecated path): use .sortOrder on each component
        val allComponents: List<SortablePlacement> = buildList {
            addAll(elements.storeCarousels)
            addAll(elements.itemCarousels)
            addAll(elements.collectionsV2)
            addAll(elements.feedPlacements)
            elements.dealCarousel?.let { add(it) }
            // etc.
        }.sortedBy { it.sortOrder ?: Int.MAX_VALUE }
        allComponents.forEachIndexed { verticalIdx, placement ->
            emitForPlacement(placement, verticalIdx, null, partial, events)
        }
    }

    // Category sections (vertical-specific sections on cross-vertical page)
    elements.categorySections?.forEach { section ->
        section.storeCarousels?.forEach { carousel ->
            emitStoreCarouselEvents(carousel, carousel.sortOrder ?: 0, section.verticalId, partial, events)
        }
    }

    // Store list (always last)
    elements.storeList.stores.forEachIndexed { idx, store ->
        events += buildStoreEvent(partial, store, elements.storeList.sortOrder ?: 0, idx, "store_list", null)
    }

    // Intermixed stores
    elements.intermixedStores?.forEachIndexed { idx, store ->
        events += buildStoreEvent(partial, store, store.sortOrder ?: 0, idx, "intermixed", null)
    }

    return events
}

private fun emitForPlacement(
    placement: SortablePlacement,
    verticalIdx: Int,
    categoryId: Long?,
    partial: CrossVerticalHomePageFeedEvent,
    events: MutableList<Event<Message>>,
) {
    when (placement) {
        is StoreCarousel -> emitStoreCarouselEvents(placement, verticalIdx, categoryId, partial, events)
        is ItemCarousel -> emitItemCarouselStoreEvents(placement, verticalIdx, categoryId, partial, events)
        is CollectionV2 -> placement.children.forEach { child ->
            emitForPlacement(child as SortablePlacement, verticalIdx, categoryId, partial, events)
        }
        is FeedPlacement -> {
            placement.store?.let { store ->
                events += buildStoreEvent(partial, store, verticalIdx, 0, "feed_placement", categoryId)
            }
        }
        // BannerCarousel, DealCarousel, GiftCardCarousel, etc. → skip (no store data)
    }
}
```

## Key field mappings (CrossVerticalHomePageFeedEvent)

| Event field | Source |
|---|---|
| `request_id` | `context.requestId` |
| `consumer_id` | `context.consumerContext.consumerId` |
| `submarket_id` | `context.geoContext.submarketId` |
| `is_dashpass_user` | `context.consumerContext.isDashpassConsumer` |
| `is_occasional_user` | `context.consumerContext.isOccasionalUser` |
| `facet_vertical_position` | `verticalIdx` (from sortedPlacements index) or `carousel.sortOrder` |
| `horizontal_position_in_facet` | store index within carousel |
| `facet_id` | `"carousel.standard:store_carousel:${carousel.id}"` |
| `facet_type` | `"store_carousel"` / `"store_list"` / `"item_carousel"` / `"feed_placement"` |
| `facet_name` | `carousel.title` |
| `facet_score` | `carousel.predictionScore` |
| `facet_score_multiplier` | `carousel.scoreMultiplier` |
| `store_id` | `store.id` |
| `store_name` | `store.name` |
| `business_id` | `store.businessId` |
| `store_primary_vertical_id` | `store.primaryVerticalIds.firstOrNull()` |
| `store_business_vertical_id` | `store.businessVerticalId` |
| `horizontal_element_score` | `store.predictionScore` |
| `predictor_names` | `store.predictorNames` |
| `model_ids` | `store.predictorModelIds` |
| `is_sponsored` | `store.isSponsored` |
| `store_is_asap_available` | `store.status?.asapAvailable` |
| `store_asap_minutes` | `store.status?.asapMinutes` |
| `consumer_preferred_store_ids` | `retrievalData.programmaticCarouselStoreFilter.preferences` |
| `consumer_preferred_category_ids` | `retrievalData.programmaticCarouselStoreFilter.preferences` |
| `total_stores` | `retrievalData.numResults` |
