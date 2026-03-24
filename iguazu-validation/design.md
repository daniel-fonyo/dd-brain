# Iguazu Event Validation — Design

## Goal

After a sandbox homepage test, validate that every carousel and store observed in the browser has a corresponding 1:1 record in Snowflake's iguazu event table. Verify scoring fields (score_modifiers, predictor_names, facet_score, etc.) are populated correctly.

## Architecture

```
/sandbox-test                              /validate-iguazu
    │                                           │
    ├─ 3 homepage loads                         ├─ Read request_ids from run dir
    ├─ Extract requestIds from pod logs         ├─ Poll Snowflake (retry if empty)
    ├─ Write to ${RUN_DIR}/meta.json            ├─ Pull full event data per request_id
    ├─ Save carousels + request_ids             ├─ Compare browser vs Snowflake 1:1
    └─ Existing audit trail                     └─ Write iguazu-validation.md to run dir
```

### Two independent skills, shared state via run directory

- `sandbox-test` captures request_ids in `meta.json` alongside existing carousel data
- `validate-iguazu` reads a run directory, extracts request_ids, queries Snowflake
- No forced idle wait — user invokes validation when ready (or skill polls with backoff)
- Re-runnable: can validate any historical run by passing its directory

---

## Request ID Capture (sandbox-test update)

### Where requestId lives in pod logs

Feed-service emits JSON-structured logs. The `requestId` field is added to every log line via `ExploreContext.customizedLogger()`:

```kotlin
// ExploreContext.kt
override fun customizedLogger(logger: KontextLogger): KontextLogger {
    return logger.withValues(
        "pipeline" to pageType,
        "consumerId" to consumerContext.consumerId,
        "requestId" to requestId,
        ...
    )
}
```

**Log format**: JSON with logstash encoder. Field name: `"requestId"` (camelCase).

### Extraction from pod logs

After each homepage load, grep the pod log for request_ids associated with consumer 757606047 (our hardcoded test consumer):

```bash
# Extract unique requestIds for our test consumer from JSON logs
grep '"consumerId":757606047' ${RUN_DIR}/feed-service.log \
  | grep -o '"requestId":"[a-f0-9-]*"' \
  | sort -u \
  | sed 's/"requestId":"//;s/"//'
```

**Timing**: Each of the 3 homepage loads generates a distinct requestId. Extract after each load by diffing the log file (note the byte offset before load, grep new lines after).

### Storage in meta.json

Add to existing meta.json structure:

```json
{
  "loads": [
    {
      "loadIndex": 1,
      "requestId": "5ccba07d-43c8-43af-860a-641761b9ba91",
      "homepageLoadMs": 4733,
      "carouselCount": 34,
      "totalStores": 286
    }
  ]
}
```

---

## Snowflake Query Design

### MCP Tool

All queries execute via the **Snowflake MCP server** tool: `mcp__snowflake__run_snowflake_query`. Pass SQL as the `statement` parameter. Returns JSON rows. If MCP connection drops, user runs `/mcp` to reconnect.

### Target table

```
IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
```

### Key column names (actual Snowflake identifiers)

- `IGUAZU_PARTITION_DATE` — VARCHAR, format `'YYYY-MM-DD'` (**NOT** `PARTITION_DATE` — that column does not exist, causes SQL error 000904)
- `IGUAZU_PARTITION_HOUR` — NUMBER (0-23, UTC)
- `CONSUMER_ID` — NUMBER
- `REQUEST_ID` — VARCHAR

### Partition filter (required for performance)

Always include BOTH `IGUAZU_PARTITION_DATE` AND `IGUAZU_PARTITION_HOUR`. Derive hour from test run timestamp (UTC). Use ±1 hour range for propagation delay.

```sql
WHERE IGUAZU_PARTITION_DATE = '2026-03-24'
  AND IGUAZU_PARTITION_HOUR BETWEEN <hour - 1> AND <hour + 1>
  AND CONSUMER_ID = 757606047
```

### Existence check (for polling)

```sql
SELECT REQUEST_ID, COUNT(*) AS ROW_COUNT
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
WHERE IGUAZU_PARTITION_DATE = '<partition_date>'
  AND IGUAZU_PARTITION_HOUR BETWEEN <hour - 1> AND <hour + 1>
  AND CONSUMER_ID = 757606047
  AND REQUEST_ID IN ('<id1>', '<id2>', '<id3>')
GROUP BY REQUEST_ID
```

### Full event pull

```sql
SELECT
    REQUEST_ID,
    CONSUMER_ID,
    FACET_VERTICAL_POSITION,
    FACET_ID,
    FACET_TYPE,
    FACET_NAME,
    STORE_ID,
    STORE_NAME,
    HORIZONTAL_POSITION_IN_FACET,
    FACET_SCORE,
    RAW_FACET_SCORE,
    FACET_SCORE_MULTIPLIER,
    HORIZONTAL_ELEMENT_SCORE,
    RAW_HORIZONTAL_ELEMENT_SCORE,
    SCORE_MODIFIERS,
    PREDICTOR_NAMES,
    MODEL_IDS,
    IS_DASHPASS_USER,
    IS_OCCASIONAL_USER,
    DD_SESSION_ID,
    DD_DEVICE_ID,
    SUBMARKET_ID,
    DISTRICT_ID,
    TIME_ZONE,
    STORE_PRIMARY_VERTICAL_ID,
    STORE_BUSINESS_VERTICAL_ID,
    STORE_IS_DASHPASS_PARTNER,
    STORE_DELIVERY_FEE_AMOUNT,
    STORE_DISTANCE_FROM_CONSUMER,
    STORE_AVERAGE_RATING,
    ENTITY_HAS_OFFER_BADGE,
    IS_SPONSORED,
    ITEM_ID,
    ITEM_NAME,
    DEAL_ID,
    DEAL_TYPE
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
WHERE IGUAZU_PARTITION_DATE = '<partition_date>'
  AND IGUAZU_PARTITION_HOUR BETWEEN <hour - 1> AND <hour + 1>
  AND CONSUMER_ID = 757606047
  AND REQUEST_ID IN ('<id1>', '<id2>', '<id3>')
ORDER BY REQUEST_ID, FACET_VERTICAL_POSITION, HORIZONTAL_POSITION_IN_FACET
```

---

## Validation Checks (1:1 matching)

### Level 1: Request-level

| Check | Browser Source | Snowflake Source |
|---|---|---|
| Consumer ID | hardcoded 757606047 | `consumer_id` |
| Request exists | pod log requestId | `request_id` row count > 0 |

### Level 2: Carousel-level (per request_id)

| Check | Browser Source | Snowflake Source |
|---|---|---|
| Carousel count | `load-N-carousels.json` → `allCarousels.length` | `COUNT(DISTINCT facet_id)` |
| Carousel IDs match | `allCarousels[].carouselId` | `DISTINCT facet_id` values |
| Carousel ordering | browser scroll position (absY rank) | `facet_vertical_position` ordering |
| Facet type | carousel ID prefix → type | `facet_type` |
| Second pass score | browser `secondPassScore` | `facet_score` |

### Level 3: Store-level (per carousel per request_id)

| Check | Browser Source | Snowflake Source |
|---|---|---|
| Store count per carousel | `allCarousels[].storeCount` | `COUNT(*)` per `facet_id` |
| Store IDs match | `allCarousels[].stores[].id` | `store_id` per `facet_id` |
| Store names match | `allCarousels[].stores[].name` | `store_name` |
| Store ranking score | `allCarousels[].stores[].score` | `horizontal_element_score` |

### Level 4: Scoring completeness (per event row)

| Field | Expectation |
|---|---|
| `score_modifiers` | Non-empty array for store_carousel facets |
| `predictor_names` | Non-empty array when scores are present |
| `model_ids` | Non-empty array when predictors are present |
| `facet_score` | > 0 for ranked carousels |
| `horizontal_element_score` | > 0 for stores with ranking |
| `raw_facet_score` | Present when facet_score present |
| `raw_horizontal_element_score` | Present when horizontal_element_score present |

### Level 5: Score modifier breakdown

For each `facet_type = 'store_carousel'` row, report which score_modifiers are present:

Expected modifiers (from IguazuEventUtil code):
- `programmatic_boost_multiplier`
- `ucb_uncertainty_score`
- `expected_vp_score` (commission)
- `dasher_cost_score`
- `user_intent_prediction_score`
- `calibration_multiplier`
- `vertical_intent_multiplier`
- `vertical_boost_weight`
- `diversity_score`
- `mor_breakdown_scores`

Store-level modifiers:
- `sr_multiplier`
- `sr_uncertainty_score`
- `ur_multiplier`
- `cold_start_boost_modifier`

---

## Polling Strategy

```
attempt 1: immediate query
attempt 2: wait 60s, query
attempt 3: wait 60s, query
attempt 4: wait 60s, query
attempt 5: wait 60s, query
```

Stop as soon as ALL request_ids have rows. Report partial results if timeout.

---

## Output: iguazu-validation.md (written to run dir)

```markdown
# Iguazu Event Validation Report

Run: <RUN_ID>
Date: <date>
Request IDs: <id1>, <id2>, <id3>

## Summary
| Check | Status |
|---|---|
| Events found for all request_ids | PASS/FAIL |
| Consumer ID matches (757606047) | PASS/FAIL |
| Carousel count matches (browser vs SF) | PASS/FAIL |
| All carousel IDs present in SF | PASS/FAIL |
| Store count per carousel matches | PASS/FAIL |
| All store IDs present in SF | PASS/FAIL |
| Score modifiers populated | PASS/FAIL |
| Predictor names populated | PASS/FAIL |

## Per-Load Comparison
### Load 1 (request_id: ...)
| Carousel | Browser Stores | SF Stores | Match | facet_score | Modifiers |
|---|---|---|---|---|---|
| Under $1 delivery fee | 8 | 8 | PASS | 0.0060 | 7/10 present |
| ... | ... | ... | ... | ... | ... |

## Score Modifier Coverage
| Modifier | Present Count | Total Rows | Coverage |
|---|---|---|---|
| programmatic_boost_multiplier | 340 | 340 | 100% |
| dasher_cost_score | 280 | 340 | 82% |
| ... | ... | ... | ... |

## Mismatches (if any)
- Store 297366 (Lucca Delicatessen) in browser carousel "Under $1" but MISSING in Snowflake
- ...
```

---

## Implementation Status

1. ~~Test Snowflake MCP connection~~ — **DONE** (2026-03-24). MCP tool `mcp__snowflake__run_snowflake_query` confirmed working. Verified query returns rows from `CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE`.
2. ~~Update `sandbox-test.md` — add requestId extraction after each load~~ — **DONE**
3. ~~Create `validate-iguazu.md` skill~~ — **DONE**
4. End-to-end test: sandbox-test → wait → validate-iguazu — **PENDING**
