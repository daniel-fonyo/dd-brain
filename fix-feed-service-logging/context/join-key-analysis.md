# Join Key Analysis: Server Iguazu Events ↔ Client Click/Impression Events

## The Problem

Our new `HomepageStoreLoadIguazuEventUtil` builds `CrossVerticalHomePageFeedEvent` from
internal models (`HomePageStoreLayoutOutputElements`), **before** serialization.

The client fires click/impression events using the `logging` maps embedded in the serialized
`FacetV2` response. Those logging maps are populated by the serializer (`StoreCarouselDataAdapter`
etc.), which runs **after** post-processing.

For any Snowflake analysis joining server-side rank/load data with client-side engagement data,
the values in both event types must match on shared keys.

---

## What the Serializer Puts in FacetV2 Logging

Every `FacetV2` the client receives has a `logging` struct. The client fires implicit analytics
events using this map. Keys relevant to joins:

### At the carousel container level:
| Key in FacetV2 logging | Value set by serializer | Source in internal model |
|---|---|---|
| `request_id` | `context.requestId` | `ExploreContext.requestId` |
| `tracking_uuid` | `context.requestId ?: UUID.randomUUID()` | same value as `request_id` |
| `container_id` | `carousel.id` (raw UUID, NOT the full facet ID) | `StoreCarousel.id` |
| `container_name` | `carousel.title` | `StoreCarousel.title` |
| `vertical_position` | added by `HomepageLoggingHelperUtil.loggingAddVerticalPosition()` | derived post-serialization (see below) |
| `facet_vertical_position` | same as `vertical_position` | same |

### At each store card level:
| Key in FacetV2 logging | Value set by serializer | Source in internal model |
|---|---|---|
| `request_id` | `context.requestId` | `ExploreContext.requestId` |
| `tracking_uuid` | same as above | same |
| `store_id` | `store.id` | `StoreEntity.id` |
| `store_name` | `store.name` | `StoreEntity.name` |
| `container_id` | `carousel.id` (raw UUID) | `StoreCarousel.id` |
| `container_name` | `carousel.title` | `StoreCarousel.title` |
| `card_position` / `horizontal_position` | store index in carousel | loop index over `carousel.stores` |
| `vertical_position` / `global_store_carousel_vertical_position` | added post-serialization | same as container (see below) |

---

## Key Insight: `request_id` == `tracking_uuid`

In `HomepageNoTileResponseSerializer`:
```kotlin
val trackingUUID = context.requestId ?: UUID.randomUUID().toString()
```

`tracking_uuid` embedded in FacetV2 logging IS `context.requestId`. So any client event that
fires with `tracking_uuid` can be joined to our server event using `request_id`.

---

## Reliable Join Keys

### Primary: `request_id` + `store_id`

```sql
server.request_id = client.request_id   -- or client.tracking_uuid
AND server.store_id = client.store_id
```

Both values are set identically in server and client events:
- `request_id`: both use `context.requestId`
- `store_id`: both come from `StoreEntity.id`

This is the most stable join. It's unaffected by position changes, carousel renames, or
serialization format differences (carousel.standard vs UC).

**Uniqueness**: After deduplication, each store appears at most once per request in the main
store carousels. So `request_id + store_id` is a unique composite key for store-level events.

Exception: the same store CAN appear in multiple item carousels (e.g. "Reorder" and "Near you"
carousels surfacing items from the same store). For item events, use `request_id + item_id`.

---

### Secondary: `request_id` + `container_id` + `horizontal_position`

```sql
server.request_id   = client.request_id
AND server.carousel_raw_id = client.container_id        -- see below
AND server.horizontal_position_in_facet = client.card_position
```

The serializer puts the **raw carousel ID** (`StoreCarousel.id`) into `container_id`,
NOT the full facet ID (`"carousel.standard:store_carousel:UUID"`).

**This means our server event needs both:**
1. `facet_id = "carousel.standard:store_carousel:${carousel.id}"` — for facet-level joins
2. `carousel_raw_id = carousel.id` — to match the `container_id` field in client events

If `CrossVerticalHomePageFeedEvent` already has a field that holds the raw carousel ID, use it.
Otherwise, the `facet_id` field contains the full prefixed string, and a Snowflake expression
can extract the raw ID: `SPLIT_PART(facet_id, ':', 3)`.

---

## Position Fields: Matching vs Mismatch Risk

### `horizontal_position_in_facet`
- **Server event**: loop index over `carousel.stores` (our approach) or `itemStoreEntities`
- **Client FacetV2 logging**: `card_position` = `carouselLoggingContext.index` = same loop index during serialization
- **Risk**: None. Both use the same loop index over the same list in the same order.

### `facet_vertical_position`
- **Server event (our approach)**: `sortedPlacements` index or `StoreCarousel.sortOrder`
- **Client FacetV2 logging**: `vertical_position` is added by `HomepageLoggingHelperUtil.loggingAddVerticalPosition()` AFTER serialization

**What does `loggingAddVerticalPosition()` actually use?**

This is the same vertical position discrepancy documented in the existing TODO:
```kotlin
// TODO: use logged vertical position for all cases after we've resolved discrepancies
// between client side vs server side vertical position
val verticalPosition = if (fromRealTimePipeline && loggedVerticalPosition != null) {
    loggedVerticalPosition
} else {
    verticalIdxOffset + verticalIdx   // counted by walking serialized FacetSections
}
```

The existing `IguazuEventUtil` acknowledges it may be different from what the client sees.
The realtime pipeline resolves this by reading `facet.logging.fieldsMap[VERTICAL_POSITION_KEY]`
(the logged position) rather than counting.

**For our approach**: `StoreCarousel.sortOrder` is set by the post-processor and is what drives
the serializer's section order. The `loggingAddVerticalPosition()` runs over the serialized
sections and assigns vertical positions by counting. If the section structure maps 1:1 to
`sortOrder` (i.e., no sections are injected between carousels that don't have a sortOrder),
these match. If extra sections (headers, banners) are interleaved between ranked carousels,
the counted position will be higher than `sortOrder`.

**Mitigation**: Don't rely on `facet_vertical_position` as a join key. Use it as an analytics
dimension only. The `request_id + store_id` join is position-independent.

---

## Facet ID: Full String vs Raw Carousel ID

The client does NOT receive nor fire events using the full facet ID string
(`"carousel.standard:store_carousel:uuid"`). The client logging uses `container_id = carousel.id`.

The full facet ID (`facet_id` in `CrossVerticalHomePageFeedEvent`) is a server-side concept used
for Iguazu event routing and Snowflake querying, not a client-emitted field. There is no direct
join on `facet_id` between server and client events.

To query "all click events for stores in a specific carousel": join on `container_id` (client)
= raw carousel UUID = `SPLIT_PART(server.facet_id, ':', 3)`.

---

## Item-Level Events

For `ItemCarouselGenerator`-based events (per item):

| Join key | Server event field | Client FacetV2 logging key |
|---|---|---|
| Primary | `request_id` + `item_id` | `request_id` + `item_id` |
| Secondary | `request_id` + `container_id_raw` + `horizontal_position_in_facet` | `request_id` + `container_id` + `card_position` |

`item_id` in the FacetV2 child logging matches `ItemStoreEntity.itemId`. Since deduplication
doesn't apply to items the way it does to stores, `request_id + item_id` is the safest primary
key for item-level analysis.

---

## Summary: What to Log in Our New Event for Maximum Joinability

| `CrossVerticalHomePageFeedEvent` field | Must match | Client event field |
|---|---|---|
| `request_id` | `context.requestId` | `request_id` / `tracking_uuid` |
| `store_id` | `store.id` (long) | `store_id` |
| `item_id` | `itemStoreEntity.itemId` | `item_id` |
| `horizontal_position_in_facet` | loop index (same order as serializer) | `card_position` / `horizontal_position` |
| `facet_id` | `"carousel.standard:store_carousel:${carousel.id}"` | not fired by client directly |
| raw carousel ID | `carousel.id` (via `SPLIT_PART(facet_id, ':', 3)`) | `container_id` |

Fields NOT needed for joins but useful for analysis:
- `facet_vertical_position` — use `sortOrder`, accept ~1-2 slot discrepancy from client's counted position
- `consumer_id`, `submarket_id`, `platform`, `session_id` — context filters, not join keys

---

## One Structural Risk: `carousel.id` Stability

The join assumes `carousel.id` (raw) in the server event == `container_id` in the client event.

This holds as long as `StoreCarouselDataAdapter.generateFacetIdFromCarouselId(carousel.id)` is
the function that writes `container_id` into the FacetV2 logging. If any adapter writes a
DIFFERENT value into `container_id` (e.g., a renamed or transformed ID), the join breaks.

Known case: **CC-format carousels** use `CCToFacetIguazuEventTransformer` which rewrites the
facet ID. The CC carousels' facet IDs are transformed from `card.store:store_cc:{id}` to
`row.store:{id}` format. The `container_id` in client logging for CC-format store cards may
reference the CC card ID, not the StoreCarousel UUID. For our approach, using `StoreCarousel.id`
directly should produce the canonical ID that the NON-CC path also uses — since CC vs non-CC is
a rendering format difference, not an ID difference. But this needs validation.

**Recommended action**: Add a validation test or shadow comparison that checks
`server.carousel_id == client.container_id` for a sample of requests before full rollout.
