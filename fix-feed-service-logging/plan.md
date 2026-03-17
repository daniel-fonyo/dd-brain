# Plan: Fix Feed-Service Homepage Logging

## Status
- [ ] Step 1: Finalize Snowflake DDL
- [ ] Step 2: New HomepageStoreLoadEventUtil (replaces LegacyHomePageIguazuEventUtil for homepage)
- [ ] Step 3: Fork in DefaultHomePagePipeline — new iguazuEventEmissionJob parallel to outputSerializerJob
- [ ] Step 4: Verify StoreEntity field availability (primaryVerticalIds, predictorNames, etc.)
- [ ] Step 5: Topic gating / config

## Key Decisions Made

### Emit CrossVerticalHomePageFeedEvent (not HomePageFeedEvent)
Use the newer structured proto. It has `request_id`, `facet_vertical_position`, `horizontal_position_in_facet`,
`store_primary_vertical_id`, `predictor_names`, `model_ids`, `is_sponsored` — all the fields we need.

### Build from internal models (not FacetSection)
Do NOT use the FacetSection approach (`IguazuEventUtil.generateXVerticalHomePageFeedSegmentEvents`).
That approach re-derives positions by counting serialized proto structure — known to produce
discrepancies (TODO comment in code). Instead, build directly from `HomePageStoreLayoutOutputElements`.

### Fork at postProcessingJob, not at serializer
Add a new workflow job `iguazuEventEmissionJob` (failOnError=false) that depends on `postProcessingJob`,
running in parallel with `outputSerializerJob`. Serialization is never blocked by logging.
See: context/pipeline-fork-approach.md

### All component types covered
storeCarousels, storeList, itemCarousels, collectionsV2/collections, feedPlacements, categorySections,
intermixedStores, reelsCarousel.decoratedStoreEntities.
Banners, dealCarousel, giftCards, verticalEntryPoints, mapCarousels skipped (no StoreEntity data).
See: context/homepage-component-coverage.md

### sortedPlacements as vertical position source (V2 path)
When `sortedPlacements.isNotEmpty()`, use the list index as `facet_vertical_position` — this is
the authoritative final order set by the post-processor.
Fall back to `component.sortOrder` for the deprecated V1 path.

---

## Background

The feed-service homepage pipeline (`getHomepageFeed`) already has a partial Iguazu logging path:

- **Event name**: `cx_home_page_feed`
- **Proto**: `HomePageFeedEvent` (fields 1–55) in `feed_service/events.proto`
- **Generator**: `LegacyHomePageIguazuEventUtil.generateHomePageFeedSegmentEvents()`
  - Emits **one event per store** across: store carousels, store list, deal carousel, collections
- **Emit point**: `DefaultHomePageResponseSerializer.recordFallbackAndEvents()` — fire-and-forget after serialization
- **Infrastructure**: `IguazuModule.sendEvent()` — wraps with topic gating, metrics, sandbox checks

The problem: the event and Snowflake table are either missing or have insufficient fields for analysis.

---

## Step 1: Snowflake Table DDL

Target table: `FEED_SERVICE.PUBLIC.CX_HOMEPAGE_STORE_LOADS`
Grain: one row per (request, store_id, carousel/dm_id)

```sql
CREATE TABLE IF NOT EXISTS FEED_SERVICE.PUBLIC.CX_HOMEPAGE_STORE_LOADS (
    -- Partitioning
    DS                                   DATE,          -- date partition (from event timestamp)
    EVENT_TIMESTAMP_UTC                  TIMESTAMP_NTZ, -- exact emit time from Iguazu

    -- Request context
    REQUEST_ID                           VARCHAR,       -- UUID; joins all rows from same request
    CONSUMER_ID                          NUMBER(38,0),
    SUBMARKET_ID                         NUMBER(38,0),
    DISTRICT_ID                          NUMBER(38,0),
    CONSUMER_LATITUDE                    FLOAT,
    CONSUMER_LONGITUDE                   FLOAT,
    PLATFORM                             VARCHAR,       -- ios/android/web
    DD_SESSION_ID                        VARCHAR,
    DD_DEVICE_ID                         VARCHAR,
    EXPERIENCE_ID                        VARCHAR,
    TIME_ZONE                            VARCHAR,
    SCHEDULED_TIME                       VARCHAR,       -- empty = ASAP
    OFFSET                               NUMBER(38,0),  -- pagination cursor offset
    TOTAL_STORES                         NUMBER(38,0),  -- total candidates from discovery

    -- Consumer state
    IS_DASHPASS_USER                     BOOLEAN,
    IS_OCCASIONAL_USER                   BOOLEAN,

    -- Display module (carousel) metadata
    DM_ID                                VARCHAR,
    DM_TYPE                              VARCHAR,       -- store_carousel / store_list / deal_carousel / collection
    DM_NAME                              VARCHAR,
    DM_DESCRIPTION                       VARCHAR,
    DM_SORT_ORDER                        NUMBER(38,0),
    DM_PREDICTION_SCORE                  FLOAT,
    DM_SCORE_MULTIPLIER                  FLOAT,

    -- Store identity
    STORE_ID                             NUMBER(38,0),
    STORE_NAME                           VARCHAR,
    BUSINESS_ID                          NUMBER(38,0),
    STORE_PRIMARY_VERTICAL_ID            NUMBER(38,0),
    STORE_BUSINESS_VERTICAL_ID           NUMBER(38,0),
    STORE_TAGS                           VARCHAR,       -- comma-separated cuisine/category tags

    -- Store position in response
    STORE_POSITION_IN_DM                 NUMBER(38,0),  -- horizontal slot index within carousel
    HORIZONTAL_MANUAL_SORT_ORDER         NUMBER(38,0),  -- -1 if not manually set

    -- Store real-time attributes
    STORE_IS_ASAP_AVAILABLE              BOOLEAN,
    STORE_ASAP_MINUTES                   NUMBER(38,0),
    STORE_NEXT_OPEN_TIME                 VARCHAR,
    STORE_NEXT_CLOSE_TIME                VARCHAR,
    STORE_LATITUDE                       FLOAT,
    STORE_LONGITUDE                      FLOAT,
    STORE_DISTANCE_FROM_CONSUMER         VARCHAR,       -- display string (e.g. "1.2 mi")

    -- Store commercial attributes
    STORE_PRICE_RANGE                    NUMBER(38,0),
    STORE_AVERAGE_RATING                 FLOAT,
    STORE_NUMBER_OF_RATINGS_DISPLAY      VARCHAR,
    STORE_IS_DASHPASS_PARTNER            BOOLEAN,
    STORE_HEADER_IMAGE_URL               VARCHAR,
    STORE_DISPLAY_DELIVERY_FEE           VARCHAR,
    DELIVERY_FEE_AMOUNT                  NUMBER(38,0),  -- cents
    MINIMUM_SUBTOTAL_AMOUNT              NUMBER(38,0),  -- cents

    -- ML / Ranking
    PREDICTION_SCORE                     FLOAT,
    PREDICTOR_NAMES                      VARCHAR,       -- comma-separated
    MODEL_IDS                            VARCHAR,       -- comma-separated
    SCORE_MODIFIERS_JSON                 VARCHAR,       -- JSON array: [{name, value}, ...]
    IS_SPONSORED                         BOOLEAN,

    -- Consumer preference signals
    ENTITY_HAS_OFFER_BADGE               BOOLEAN,
    CONSUMER_PREFERRED_STORE_IDS         VARCHAR,
    CONSUMER_PREFERRED_CATEGORY_IDS      VARCHAR,
    CS_ST_CUISINE_AFFINITY_SIMILARITY    FLOAT,
    PERCENTAGE_MATCH                     FLOAT
)
CLUSTER BY (DS, SUBMARKET_ID, CONSUMER_ID);
```

---

## Step 2: Proto Field Additions

Add to `HomePageFeedEvent` in `feed_service/events.proto` (continuing from field 56):

```protobuf
string request_id = 56;
int64 store_primary_vertical_id = 57;
int64 store_business_vertical_id = 58;
repeated string predictor_names = 59;
repeated string model_ids = 60;
message ScoreModifier {
  string name = 1;
  double value = 2;
}
repeated ScoreModifier score_modifiers = 61;
bool is_dashpass_user = 62;
bool is_occasional_user = 63;
bool is_sponsored = 64;
string store_tags = 65;
int64 horizontal_manual_sort_order = 66;
// Next ID: 67
```

---

## Step 3: LegacyHomePageIguazuEventUtil Changes

### On the shared `partial` builder (applies to all events in the request):
```kotlin
.setRequestId(homepageContext.requestId ?: UUID.randomUUID().toString())
.setIsDashpassUser(homepageContext.consumerContext.isDashpassConsumer)
// is_occasional_user if available
```

### Per-store (in both carousel loop and store list loop):
```kotlin
.setStorePrimaryVerticalId(store.primaryVerticalId ?: 0L)
.setStoreBusinessVerticalId(store.businessVerticalId ?: 0L)
.setStoreTags(store.tags.joinToString(","))
.setIsSponsored(store.isSponsored ?: false)
.setHorizontalManualSortOrder(store.manualSortOrder ?: -1L)
.addAllPredictorNames(store.rankingFeatures?.predictorNames ?: emptyList())
.addAllModelIds(store.rankingFeatures?.modelIds ?: emptyList())
.addAllScoreModifiers(store.rankingFeatures?.scoreModifiers?.map {
    Events.HomePageFeedEvent.ScoreModifier.newBuilder()
        .setName(it.name).setValue(it.value).build()
} ?: emptyList())
```

---

## Step 4: StoreEntity Field Availability (TBD)

Need to verify that `StoreEntity` (the model in `HomePageStoreLayoutOutputElements`) carries:
- `primaryVerticalId`
- `businessVerticalId`
- `tags`
- `isSponsored`
- `rankingFeatures` (predictorNames, modelIds, scoreModifiers)
- `manualSortOrder`

If any are missing from `StoreEntity`, trace back through:
`DiscoveryStore` → decoration stages → `LiteStoreCollection` → layout → `StoreEntity`

---

## Step 5: Topic Gating

Verify `PlatformRuntimeUtil.shouldEnableIguazuTopic("cx_home_page_feed")` returns `true` in prod.
Check the dynamic value / LaunchDarkly flag that controls this.
