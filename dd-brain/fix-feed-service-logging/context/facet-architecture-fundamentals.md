# Facet Architecture — First Principles

## What is a Facet?

A **facet** (`FacetV2`) is a single **renderable unit** in the feed response. It is a universal
container that tells the client both WHAT to show and HOW to show it, without the server needing
to know the concrete UI component implementation.

Think of it as a self-describing render instruction:
```
FacetV2 {
  id       → identity: what content is this, rendered by what engine
  component → which rendering engine to invoke on the client
  text      → display strings (title, subtitle, description)
  images    → visual assets
  custom    → arbitrary JSON payload, contract specific to the component type
  logging   → analytics data emitted implicitly when the facet is viewed/clicked
  children  → nested FacetV2s (e.g. the stores inside a carousel)
  layout    → padding, grid sizing
  style     → colors, fonts, borders
}
```

The server describes structure and content. The client renders it using the component registry.

---

## What is a FacetSection?

A **FacetSection** is a horizontal slice of the page — a logical grouping of one or more facets that
belong together. It has an `id` (e.g. `"global_carousels"`, `"banners"`, `"store_list_header"`),
and a `body: repeated FacetV2` which is the actual content.

```
GetFacetFeedResponseV2
  └── body: repeated FacetSection
        ├── FacetSection(id="banners")        → 1 facet: a banner
        ├── FacetSection(id="global_carousels")→ N facets: one per carousel
        ├── FacetSection(id="store_feed")     → M facets: one per store row
        └── FacetSection(id="category_section;1") → N facets: carousels for vertical 1
```

Sections are page-layout groupings, not semantic groupings — they determine how the page sections
are separated, padded, and scrolled. The logging system uses `sectionId` to track which section
of the page an event came from.

---

## The Parent → Child Relationship

A carousel facet has two layers:

```
FacetSection(id="global_carousels")
  body[0]: FacetV2(id="carousel.standard:store_carousel:uuid")   ← CAROUSEL CONTAINER
    text.title = "Best Rated Near You"
    logging    = { carousel_id, carousel_name, prediction_score, ... }
    children[]: FacetV2                                           ← CAROUSEL ITEMS
      [0]: FacetV2(id="card.store:store:12345")                   ← STORE CARD
        text.title = "Pizza Palace"
        logging    = { store_id=12345, store_name, prediction_score, position=0, ... }
      [1]: FacetV2(id="card.store:store:67890")
        ...
```

- The **body facet** (level 1) is the carousel container. Its `logging` carries carousel-level analytics.
- The **children facets** (level 2) are the individual items (store cards, item cards, tile cards).
  Each child's `logging` carries item-level analytics.

This means `FacetV2.logging` is the analytics contract for that unit — the client fires an implicit
impression/click event using whatever fields are in that logging map. The keys are strings, the
values are arbitrary JSON values (`google.protobuf.Struct`).

**Why is logging in the children, not in custom?**
`custom` is for rendering data (what to display). `logging` is for analytics (what to track).
The distinction matters because the client fires logging events without needing to understand the
business semantics of `custom`.

---

## The Facet ID Format: `rendering_engine:content_type:content_id`

Every facet ID encodes three things, colon-separated:

```
carousel.standard  :  store_carousel  :  3f2a-4b1c-uuid
      [1]                 [2]                 [3]
```

**[1] Rendering engine / component family** — which client component to use:
| Prefix | What it means |
|---|---|
| `carousel.standard` | Legacy standard carousel renderer (horizontal scroll, DoorDash-specific) |
| `cx.common.universal_carousel` | Universal Carousel framework (flexible, cross-vertical component) |
| `card.store` | A single store card |
| `card.tile` | A navigation/category tile |
| `row.store` | A full-width store row (for list view) |
| `merchandisingunit_component` | Ads/merchandising placement |
| `collection.standard` | A curated collection wrapper |

**[2] Content type** — what kind of content lives inside:
| Content type | Meaning |
|---|---|
| `store_carousel` | A set of stores displayed horizontally |
| `item_carousel` | A set of items displayed horizontally |
| `deal_carousel` | A set of deal offers |
| `tile_collection` | Navigation/category tiles (no individual stores) |
| `store_collection` | Curated store tiles (tile-based store collection) |
| `reels` | Video reels / announcements |
| `store` | A single store (leaf node, used in `card.store:store:ID`) |

**[3] Content ID** — the specific instance:
- For carousels: UUID of the carousel, often matching `StoreCarousel.id` or `ItemCarousel.id`
- For store rows: the `storeId`
- For tile carousels: `tile_carousel` (static, no UUID needed since it's per-page)

### Why structure it this way?

The three-part ID is a **stable contract between server and client**:
- The client uses part [1] to look up the component in its component registry
- The client uses part [2] to know how to interpret `custom` and `children`
- The client uses part [3] for deduplication, deep linking, and logging

The server doesn't need to know what version of the client component is running — it just needs to
produce a valid ID. The client is responsible for mapping `carousel.standard:store_carousel:*` to
its concrete UI implementation.

---

## Two Rendering Architectures: carousel.standard vs cx.common.universal_carousel

This is the most important distinction for understanding why there are two sets of generators in
`ContainerEventsGenerator`.

### `carousel.standard` (legacy, ~2019–2022)
- DoorDash-native carousel component
- The server produces `FacetV2.children[]` — a flat list of child facets
- Each child is `card.store:store:ID` or `card.item_square:item:ID`
- The client has hardcoded knowledge of what a `carousel.standard:store_carousel` looks like
- Logging data lives in `facet.logging.fieldsMap` (flat key-value struct)
- Data lives in `facet.children[idx].logging.fieldsMap` per item

```
carousel.standard:store_carousel:uuid
  logging: { carousel_id, carousel_name, prediction_score, ... }
  children[]:
    card.store:store:12345
      logging: { store_id, store_name, asap_minutes, ... }
    card.store:store:67890
      ...
```

### `cx.common.universal_carousel` (modern, ~2022+)
- Cross-platform universal component system
- More flexible — the server produces a structured `custom` payload instead of `children`
- The component system (`cx.common.universal_carousel`) is a general-purpose renderer
- Data lives in `facet.custom.fieldsMap["logging"]` (carousel level) and inside structured
  `subcomponents` → `dynamic_height_content` → `contents[]` (item level)
- Supports richer layout options, better multi-platform compatibility
- The ID format: `cx.common.universal_carousel:store_carousel:uuid` or
  `cx.common.universal_carousel:tile_carousel:uuid`

```
cx.common.universal_carousel:store_carousel:uuid
  custom: {
    logging: { carousel_id, carousel_name, ... }
    subcomponents: [
      { standard_header: { title: { default: { value: "Best Rated" } } } },
      { dynamic_height_content: {
          contents: [
            { store_card: { logging: { store_id, ... } } },
            { store_card: { logging: { store_id, ... } } },
          ]
      }}
    ]
  }
```

**Why this matters for our approach:**
In the FacetSection path, you need different parsing code for each engine:
- `carousel.standard` → `facet.children[idx].logging.fieldsMap`
- `cx.common.universal_carousel` → `facet.custom["subcomponents"][1]["dynamic_height_content"]["contents"][idx]["store_card"]["logging"]`

In the internal model approach, BOTH `StoreCarousel` (carousel.standard) and
`UniversalStoreCarousel` (cx.common.universal_carousel) are just `StoreCarousel` objects with
`stores: List<StoreEntity>`. The serialization complexity is irrelevant — we work directly from
the model.

---

## What is `cx.common.universal_carousel:tile_carousel:*`?

A **tile carousel** is a horizontal row of navigation/category tiles — e.g., "Pizza", "Grocery",
"Sushi", "Chinese". Each tile is a category shortcut, not an individual store.

```
cx.common.universal_carousel:tile_carousel:beer_stores
  custom: {
    logging: { carousel_id: "beer_stores", carousel_name: "Beer & Wine" }
    subcomponents: [
      { standard_header: { title: { default: { value: "Categories" } } } },
      { horizontal_linear_content: {
          rows: [
            { items: [
              { category_card: {
                  logging: {
                    tile_id: "pizza", tile_name: "Pizza",
                    tile_store_ids: "1,2,3",    ← comma-separated store IDs (NOT individual stores)
                    vertical_id: 4              ← the category/vertical
                  }
              }},
              { category_card: { logging: { tile_id: "grocery", ... } } },
            ]}
          ]
      }}
    ]
  }
```

**Key insight**: A tile's `tile_store_ids` is a list of store IDs ASSOCIATED with the tile category,
not a set of individual store cards. The tile represents a category/destination. Clicking a tile
navigates to a vertical page for that category — there is no individual store being "loaded" for the
tile itself.

This is exactly why `TileCollectionGenerator` and `UniversalTileCarouselGenerator` emit events with
`storeId = 0` — there is no single `storeId` to log. And why these are correctly skipped for a
store-load logging table.

---

## How the logging data flows

```
[Server pipeline]
StoreCarousel
  └── StoreEntity (store, predictionScore, rankingFeatures, ...)
          │
          ▼ (serializer: StoreCarouselDataAdapter)
[FacetV2 logging map]
  "carousel.standard:store_carousel:uuid"
    logging: {
      container_score: carousel.predictionScore,
      container_name:  carousel.title,
      ...
    }
    children[idx]:
      logging: {
        store_id:                store.id,
        store_name:              store.name,
        store_prediction_score:  store.predictionScore,
        store_primary_vertical_id: store.primaryVerticalIds.first(),
        ...
      }

[IguazuEventUtil / ContainerEventsGenerator]
  reads back from logging map → builds CrossVerticalHomePageFeedEvent proto

[Our new approach]
  reads directly from StoreEntity → builds CrossVerticalHomePageFeedEvent proto
  (no round-trip through logging map)
```

The logging map is an intermediate encoding for the client to fire implicit analytics events.
For server-side Iguazu logging, it's an unnecessary detour — we already have the source data.

---

## Quick reference: what internal model maps to what facet ID

| Internal model | Facet ID pattern | Notes |
|---|---|---|
| `StoreCarousel` (standard) | `carousel.standard:store_carousel:{id}` | Standard rendering engine |
| `StoreCarousel` (universal) | `carousel.universal:store_carousel:{id}` | UC rendering engine (still StoreCarousel in model) |
| `StoreEntity` (in list) | `row.store:store:{storeId}` or `card.store:store:{storeId}` | Individual store row in store list |
| `ItemCarousel` (standard) | `carousel.standard:item_carousel:{id}` | |
| `ItemCarousel` (UC) | `cx.common.universal_carousel:item_carousel:{id}` | |
| `ItemCarousel` (multi-MX) | `carousel.standard:deals_v2_carousel` | Grouped item carousel |
| `CollectionV2` (store children) | `collection.standard:store_collection:{id}` → children: `carousel.standard:store_carousel:{id}` | Two-level nesting |
| `FeedPlacement` (standard) | `merchandisingunit_component:{id}` | Ads placement |
| `FeedPlacement` (UC store) | `merchandisingunit_component_store_carousel_uc:{id}` | UC ads placement |
| `FeedPlacement` (standalone) | `merchandisingunit_collection_banner:{id}` | Single-store spotlight |
| `AnnouncementCollection` | `carousel.standard:reels:reels_carousel` | Reels/videos |
| Tile collections | `custom_tile_collection:*` or `carousel.standard:tile_collection:*` or `cx.common.universal_carousel:tile_carousel:*` | Category nav tiles — NO individual stores |
| `DealCarousel` | `carousel.standard:deal_carousel:{id}` | Deal offers |
