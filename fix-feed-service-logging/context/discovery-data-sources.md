# Discovery Data Sources

> Source: https://doordash.atlassian.net/wiki/spaces/Eng/pages/2955018904/Discovery+Data+Sources

Tables available for discovery-related analysis.

---

## Server-side Tracking Events

### `IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE`

For each request to `getHomePageFacetFeed`, we emit all store / item / deal carousel and store feed events. Granularity is at the store / item level.

#### Common Fields

| Field | Example | Description |
|---|---|---|
| `facet_id` | `carousel.standard:store_carousel:08ec949a-d36c-456a-bef2-707bae2f0859` | Unit representation in Lego world. Carousel IDs are prefixed with `carousel.standard:store_carousel:`. |
| `facet_type` | `store_carousel` | `store_list` (All Stores), `store_carousel`, `item_carousel`, `deal_list` (Offers For You) |
| `facet_name` | `Flowers with $0 Delivery Fee` | Translated carousel display title |
| `facet_vertical_position` | `0` | Vertical position on homepage, starting at 0; counts only the 4 facet_types above |
| `facet_score` | `0.23` | Final score from Universal Ranker (after EnE and FVM multipliers) |
| `horizontal_position_in_facet` | `1` | Horizontal position of element within the facet |
| `horizontal_element_score` | `0.03` | Score from Store Ranker stage |
| `score_modifiers` | `[{"name": "programmatic_boost_multiplier", "value": 1.0}, {"name": "ucb_uncertainty_score", "value": 0.01}]` | Score modifiers applied to `facet_score`. `programmatic_boost_multiplier` = FVM, `ucb_uncertainty_score` = EnE |
| `model_ids` | `["universal_ranker_v3_calibrated_composite_2022_05_18_08_24_45", "programmatic_boosting_multiplier_v2_3", "store_distance_ranking_5"]` | Sibyl model IDs. First = UR model, second = FVM model, third = store ranker model |

---

### `IGUAZU.SERVER_EVENTS_PRODUCTION.ARGO_SEARCH_DISCOVERY_QUERY_LOGGER`

Tracks results emitted from FPR (first pass ranker).

```sql
SELECT SEARCH_REQUEST, USE_CASE_ID, SEARCH_RESULT_SUMMARY, *
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.ARGO_SEARCH_DISCOVERY_QUERY_LOGGER
WHERE
    IGUAZU_SENT_AT > current_date - 1
    AND CONSUMER_ID = '1234567890'
    AND (
        USE_CASE_ID = 'discovery/store_search'
        OR USE_CASE_ID = 'discovery/store_discovery_feed'
        OR USE_CASE_ID = 'discovery/store_discovery'
        OR USE_CASE_ID = 'discovery/fpr_store_discovery'
        OR USE_CASE_ID = 'discovery/nv_store_search'
        OR USE_CASE_ID = 'discovery/nv_store_search_feed'
    )
ORDER BY IGUAZU_SENT_AT DESC
```

---

### `IGUAZU.SERVER_EVENTS_PRODUCTION.HOME_PAGE_STORE_RANKER_COMPACT_EVENT`

Tracks results emitted after the Store Ranker stage.

---

## Client-side Tracking Events

### `IGUAZU.CONSUMER.M_CARD_VIEW`

Captures user impression events from mobile (use `IGUAZU.CONSUMER.CARD_VIEW` for web). Covers homepage, Search, filters, store page, and offers tab.

**Filter to homepage only:**
```sql
container = 'cluster' AND page = 'explore_page'
```

**Joining to `CX_CROSS_VERTICAL_HOME_PAGE_FEED`** (no unique ID — join on composite key):
```sql
    a.dd_session_id   = b.dd_session_id
AND a.dd_device_id    = b.dd_device_id
AND a.consumer_id     = b.consumer_id
AND a.store_id        = b.store_id
AND a.carousel_name   = b.facet_name
AND a.card_position   = b.horizontal_position_in_facet
AND a.vertical_position = b.facet_vertical_position
```

#### Common Fields

| Field | Example | Description |
|---|---|---|
| `container_id` | `08ec949a-d36c-456a-bef2-707bae2f0859` | Carousel ID (when `container = 'cluster'`) |
| `vertical_position` | `2.0` | Vertical position of the container; equivalent to `facet_vertical_position` in server event |
| `other_properties['container_score']` | `"0.23"` | Universal Ranker score (after EnE + FVM multipliers) |
| `card_position` | `1.0` | Equivalent to `horizontal_position_in_facet` |
| `store_prediction_score` | `0.03` | Store Ranker score |

---

### `IGUAZU.CONSUMER.M_CARD_CLICK`

Captures user click events from mobile (use `IGUAZU.CONSUMER.CARD_CLICK` for web). Same homepage filter as `M_CARD_VIEW`.

**Joining `M_CARD_VIEW` to `M_CARD_CLICK`** (no unique ID — join on composite key):
```sql
    dd_session_id, dd_device_id, consumer_id,
    store_id, carousel_name, card_position, vertical_position
```

Common fields: same as `M_CARD_VIEW`.

---

### `SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.M_CHECKOUT_PAGE_SYSTEM_CHECKOUT_SUCCESS`

Captures order placement success events from mobile. Use `SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.SYSTEM_CHECKOUT_SUCCESS` for web.

---

## Business Vertical IDs

Check `business_vertical_id` from `DOORDASH_MERCHANT.PUBLIC.MAINDB_BUSINESS`.
- `0` or `null` = typically Rx
- NV business lines: join with `PRODDB.STATIC.CNG_BUSINESS_VERTICAL_ID`
