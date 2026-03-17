# Order Rate & GMV Impact Analysis
## Store Carousel Iguazu Event Regression (Jan 21 – Mar 10, 2026)

---

## Simple GOV Estimation (Start Here)

Two fast queries. No joins. No per-request grouping.

**Query A — GOV by period (checkout table only)**

```sql
SELECT
    CASE
        WHEN DATE(timestamp) BETWEEN '2026-01-15' AND '2026-01-19' THEN 'baseline'
        WHEN DATE(timestamp) BETWEEN '2026-01-21' AND '2026-03-10' THEN 'regression'
    END                               AS period,
    COUNT(DISTINCT order_cart_id)     AS orders,
    SUM(subtotal) / 100.0            AS gov_usd,
    AVG(subtotal) / 100.0            AS avg_order_value_usd
FROM (
    SELECT timestamp, order_cart_id, subtotal, consumer_id
    FROM SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.M_CHECKOUT_PAGE_SYSTEM_CHECKOUT_SUCCESS
    WHERE DATE(timestamp) BETWEEN '2026-01-15' AND '2026-03-10'
    UNION ALL
    SELECT timestamp, order_cart_id, subtotal, consumer_id
    FROM SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.SYSTEM_CHECKOUT_SUCCESS
    WHERE DATE(timestamp) BETWEEN '2026-01-15' AND '2026-03-10'
) combined
GROUP BY 1
HAVING period IS NOT NULL
ORDER BY 1;
```

**Query B — Baseline request count (5 days only, fast)**

```sql
SELECT APPROX_COUNT_DISTINCT(request_id) AS baseline_requests
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
WHERE iguazu_partition_date BETWEEN '2026-01-15' AND '2026-01-19';
```

For regression-period requests: use the sum from Appendix A2 of the regression doc
(~2.77B total across Jan 21 – Mar 10) — no need to recompute.

**Impact formula**

```
baseline_gov_per_request  = baseline_gov_usd  / baseline_requests
regression_gov_per_request = regression_gov_usd / 2,770,000,000   -- from appendix A2

lost_gov_per_request      = baseline_gov_per_request - regression_gov_per_request
estimated_lost_gov        = lost_gov_per_request × 2,770,000,000

lost_orders               = (baseline_orders/baseline_requests - regression_orders/2,770,000,000)
                            × 2,770,000,000
```

**Caveat:** This is a before/after comparison — the delta includes seasonality (Jan vs Feb/Mar
demand differs naturally). It is an upper bound on regression-attributable loss, not a
causal estimate. Use Queries 4/5 if you need the treatment/control split.

---

## Causal Mechanism

The regression was a logging regression — store carousels were still rendered. The path to CX
user impact runs through ML model quality:

```
Logging regression
  → incomplete store carousel training data (1.85 vs 13.54 events/request on broken path)
  → models retrained on bad data serve degraded rankings
  → users see less relevant stores in carousels
  → lower CTR on store carousels
  → lower order conversion and/or smaller baskets
```

Impact would **not** be immediate on Jan 21; it would accumulate gradually as ML models
(store ranker, universal ranker, blender weights) were retrained on degraded data.

The LCM data in Appendix A3 of the regression doc gives us a natural experiment:
- **Broken path**: requests with LCM carousels → `StoreCarouselMapper` → 1.85 store_carousel events/request
- **Healthy path**: requests without LCM → old code → 13.54 store_carousel events/request

This LCM split can be used as treatment/control to isolate causal impact.

---

## Baseline & Regression Windows

| Period | Dates | Days |
|---|---|---|
| Baseline | Jan 15–19, 2026 | 5 (pre-regression, clean data) |
| Regression | Jan 21 – Mar 10, 2026 | 49 |

---

## Schema Notes (from M_CARD_VIEW sample records)

Confirmed flat columns: `timestamp`, `consumer_id`, `container`, `page`, `container_id`,
`carousel_name`, `card_position`, `vertical_position`, `store_id`, `store_prediction_score`.

`container_score` and `container_predictor_names` live inside the `other_properties` variant
column — access via `other_properties['container_score']::FLOAT` in Snowflake.

Session ID on client events uses an `sx_` prefix (e.g. `sx_1853ed7f-...`); the server event
`dd_session_id` does not — strip prefix if joining across tables.

Web tables (`IGUAZU.CONSUMER.CARD_VIEW` / `CARD_CLICK`) column names assumed to match
mobile — verify before running.

---

## Query 1 — Store Carousel CTR (Primary CX Signal)

Click-through rate on store carousel cells is the most direct measure of ranking quality
degradation visible to users. Uses `M_CARD_VIEW` / `M_CARD_CLICK` (mobile) and
`CARD_VIEW` / `CARD_CLICK` (web). Filtered to homepage (`page = 'explore_page'`) carousel
(`container = 'cluster'`) store cards (`store_id IS NOT NULL`) to exclude item carousel rows.

```sql
WITH

-- Mobile impressions: homepage store carousel cells only
mobile_views AS (
    SELECT
        DATE(timestamp)              AS ds,
        consumer_id,
        container_id,                -- carousel UUID; joins to M_CARD_CLICK
        carousel_name,
        card_position,               -- horizontal slot index (= horizontal_position_in_facet)
        vertical_position,           -- carousel row on page  (= facet_vertical_position)
        store_id,
        store_prediction_score,                             -- flat column, store ranker score
        other_properties['container_score']::FLOAT AS container_score  -- JSON variant, UR final score
    FROM IGUAZU.CONSUMER.M_CARD_VIEW
    WHERE DATE(timestamp) BETWEEN '2026-01-15' AND '2026-03-10'
      AND container  = 'cluster'       -- carousel type
      AND page       = 'explore_page'  -- homepage only
      AND store_id   IS NOT NULL        -- store cards only (excludes item carousel rows)
),

-- Web impressions (column names assumed to match mobile — verify)
web_views AS (
    SELECT
        DATE(timestamp)              AS ds,
        consumer_id,
        container_id,
        carousel_name,
        card_position,
        vertical_position,
        store_id,
        store_prediction_score,
        other_properties['container_score']::FLOAT AS container_score
    FROM IGUAZU.CONSUMER.CARD_VIEW
    WHERE DATE(timestamp) BETWEEN '2026-01-15' AND '2026-03-10'
      AND container  = 'cluster'
      AND page       = 'explore_page'
      AND store_id   IS NOT NULL
),

all_views AS (
    SELECT * FROM mobile_views
    UNION ALL
    SELECT * FROM web_views
),

-- Mobile clicks: same filters
mobile_clicks AS (
    SELECT
        DATE(timestamp)              AS ds,
        consumer_id,
        container_id,
        card_position,
        store_id
    FROM IGUAZU.CONSUMER.M_CARD_CLICK
    WHERE DATE(timestamp) BETWEEN '2026-01-15' AND '2026-03-10'
      AND container  = 'cluster'
      AND page       = 'explore_page'
      AND store_id   IS NOT NULL
),

-- Web clicks
web_clicks AS (
    SELECT
        DATE(timestamp)              AS ds,
        consumer_id,
        container_id,
        card_position,
        store_id
    FROM IGUAZU.CONSUMER.CARD_CLICK
    WHERE DATE(timestamp) BETWEEN '2026-01-15' AND '2026-03-10'
      AND container  = 'cluster'
      AND page       = 'explore_page'
      AND store_id   IS NOT NULL
),

all_clicks AS (
    SELECT * FROM mobile_clicks
    UNION ALL
    SELECT * FROM web_clicks
),

-- Daily aggregates
daily_views AS (
    SELECT DATE(ds) AS ds, COUNT(*) AS impressions
    FROM all_views
    GROUP BY 1
),

daily_clicks AS (
    SELECT DATE(ds) AS ds, COUNT(*) AS clicks
    FROM all_clicks
    GROUP BY 1
),

daily_ctr AS (
    SELECT
        v.ds,
        v.impressions,
        c.clicks,
        c.clicks * 1.0 / NULLIF(v.impressions, 0)  AS ctr,
        CASE
            WHEN v.ds BETWEEN '2026-01-15' AND '2026-01-19' THEN 'baseline'
            WHEN v.ds BETWEEN '2026-01-21' AND '2026-03-10' THEN 'regression'
        END AS period
    FROM daily_views v
    JOIN daily_clicks c ON v.ds = c.ds
)

SELECT
    period,
    COUNT(*)                                      AS days,
    SUM(impressions)                              AS total_impressions,
    SUM(clicks)                                   AS total_clicks,
    ROUND(SUM(clicks) * 1.0
      / NULLIF(SUM(impressions), 0),          6)  AS period_ctr
FROM daily_ctr
WHERE period IS NOT NULL
GROUP BY 1
ORDER BY 1;
```

---

## Query 2 — Session-Level Conversion Rate (Homepage View → Order)

Compares how often a homepage session converted to an order. Includes both mobile and web
checkout events.

> **Verify before running:** `order_cart_id` (unique order key) and `subtotal` (cents) in
> the checkout tables — field names may differ. `timestamp` is confirmed from the M_CARD_VIEW
> sample as the standard event time column.

```sql
WITH

-- Homepage sessions (server side — one row per request dedupe)
hp_sessions AS (
    SELECT
        iguazu_partition_date        AS ds,
        APPROX_COUNT_DISTINCT(request_id)  AS requests
    FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
    WHERE iguazu_partition_date BETWEEN '2026-01-15' AND '2026-03-10'
    GROUP BY 1
),

-- Orders: mobile + web unified
all_orders AS (
    -- Mobile
    SELECT
        DATE(timestamp)              AS ds,
        order_cart_id,
        subtotal
    FROM SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.M_CHECKOUT_PAGE_SYSTEM_CHECKOUT_SUCCESS
    WHERE DATE(timestamp) BETWEEN '2026-01-15' AND '2026-03-10'

    UNION ALL

    -- Web/desktop
    SELECT
        DATE(timestamp)              AS ds,
        order_cart_id,
        subtotal
    FROM SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.SYSTEM_CHECKOUT_SUCCESS
    WHERE DATE(timestamp) BETWEEN '2026-01-15' AND '2026-03-10'
),

daily_orders AS (
    SELECT
        ds,
        COUNT(DISTINCT order_cart_id)        AS orders,
        AVG(subtotal) / 100.0                AS avg_order_value_usd,
        SUM(subtotal) / 100.0                AS total_gmv_usd
    FROM all_orders
    GROUP BY 1
),

daily_metrics AS (
    SELECT
        s.ds,
        s.requests,
        o.orders,
        o.avg_order_value_usd,
        o.total_gmv_usd,
        o.orders * 1.0 / NULLIF(s.requests, 0)  AS order_rate,
        CASE
            WHEN s.ds BETWEEN '2026-01-15' AND '2026-01-19' THEN 'baseline'
            WHEN s.ds BETWEEN '2026-01-21' AND '2026-03-10' THEN 'regression'
        END AS period
    FROM hp_sessions s
    JOIN daily_orders o ON s.ds = o.ds
)

SELECT
    period,
    COUNT(*)                                  AS days,
    SUM(requests)                             AS total_requests,
    SUM(orders)                               AS total_orders,
    ROUND(SUM(orders) * 1.0
      / NULLIF(SUM(requests), 0), 6)          AS period_order_rate,
    ROUND(SUM(total_gmv_usd)
      / NULLIF(SUM(orders), 0), 2)            AS period_avg_order_value_usd,
    ROUND(SUM(total_gmv_usd), 0)              AS total_gmv_usd
FROM daily_metrics
WHERE period IS NOT NULL
GROUP BY 1
ORDER BY 1;
```

---

## Query 3 — Impact Estimation (Rate + Size Effects)

Uses period summaries from Query 2 to estimate lost orders and lost GMV, decomposed into
a rate effect (fewer conversions) and a size effect (smaller baskets).

```sql
WITH

-- (paste hp_sessions, all_orders, daily_orders, daily_metrics CTEs from Query 2 above)

period_agg AS (
    SELECT
        period,
        SUM(requests)                                           AS total_requests,
        SUM(orders)                                             AS total_orders,
        SUM(orders) * 1.0 / NULLIF(SUM(requests), 0)           AS period_order_rate,
        SUM(total_gmv_usd) / NULLIF(SUM(orders), 0)            AS period_avg_order_value_usd
    FROM daily_metrics
    WHERE period IS NOT NULL
    GROUP BY 1
),

baseline   AS (SELECT * FROM period_agg WHERE period = 'baseline'),
regression AS (SELECT * FROM period_agg WHERE period = 'regression')

SELECT
    -- Rates
    ROUND(b.period_order_rate,             6)  AS baseline_order_rate,
    ROUND(r.period_order_rate,             6)  AS regression_order_rate,
    ROUND(b.period_order_rate
        - r.period_order_rate,             6)  AS order_rate_delta,

    -- Avg order value (basket size)
    ROUND(b.period_avg_order_value_usd,    2)  AS baseline_avg_order_value_usd,
    ROUND(r.period_avg_order_value_usd,    2)  AS regression_avg_order_value_usd,
    ROUND(b.period_avg_order_value_usd
        - r.period_avg_order_value_usd,    2)  AS avg_order_value_delta_usd,

    -- Lost orders (rate effect)
    ROUND((b.period_order_rate - r.period_order_rate)
        * r.total_requests,                0)  AS estimated_lost_orders,

    -- Lost GMV: rate effect (fewer conversions × baseline basket)
    ROUND((b.period_order_rate - r.period_order_rate)
        * r.total_requests
        * b.period_avg_order_value_usd,    0)  AS lost_gmv_rate_effect_usd,

    -- Lost GMV: size effect (smaller baskets × actual orders placed)
    ROUND(r.total_orders
        * (b.period_avg_order_value_usd
        -  r.period_avg_order_value_usd),  0)  AS lost_gmv_size_effect_usd,

    -- Total estimated lost GMV
    ROUND(
        (b.period_order_rate - r.period_order_rate)
            * r.total_requests * b.period_avg_order_value_usd
        + r.total_orders
            * (b.period_avg_order_value_usd - r.period_avg_order_value_usd),
        0
    )                                          AS total_estimated_lost_gmv_usd

FROM baseline b
CROSS JOIN regression r;
```

---

## Query 4 — Andromeda Path as Treatment/Control (Causal Estimate)

The broken path is identifiable directly in `CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE`:

- **Broken path** (`isAndromedaStoreCarouselEnabled = true`): request routed through
  `StoreCarouselMapper` → wrong facet IDs → **≤3 `store_carousel` rows emitted** (avg 1.85)
- **Healthy path** (`isAndromedaStoreCarouselEnabled = false`): old code path →
  correct facet IDs → **~13–15 `store_carousel` rows per request**
- **Ambiguous (4–9 events)**: excluded — clear gap between the two populations

### Sampling strategy

The full 49-day scan + `GROUP BY request_id` is expensive. Use **1% consumer sampling**:
filter both tables to `MOD(consumer_id, 100) < 1` before any joins. This keeps all rows
for each sampled consumer consistent across server events and checkout tables. Multiply
count outputs by 100 to scale back to full population. Rate outputs (order_rate,
avg_order_value) need no scaling.

> **Alternative (faster, no scaling):** Replace the date range with 2–3 representative
> days per DV phase (e.g. `'2026-01-25'`, `'2026-02-14'`, `'2026-03-05'`) and compare
> broken vs healthy within each day. Good for a quick sanity check.

```sql
WITH

-- 1% consumer sample — consistent across both tables, no scaling needed for rates
sampled_consumers AS (
    SELECT DISTINCT consumer_id
    FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
    WHERE iguazu_partition_date BETWEEN '2026-01-21' AND '2026-03-10'
      AND MOD(consumer_id, 100) < 1   -- 1% sample; increase to < 10 for 10%
),

-- Classify each request as broken or healthy path.
-- Broken: ≤3 store_carousel events (avg 1.85 on broken path).
-- Healthy: ≥10 store_carousel events (avg 13.54 on healthy path).
-- 4–9: ambiguous, excluded.
request_path AS (
    SELECT
        e.request_id,
        e.consumer_id,
        e.iguazu_partition_date                                    AS ds,
        COUNT(CASE WHEN e.facet_type = 'store_carousel' THEN 1 END) AS store_carousel_count,
        COUNT(CASE WHEN e.facet_type = 'store_list'     THEN 1 END) AS store_list_count
    FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE e
    JOIN sampled_consumers s ON e.consumer_id = s.consumer_id
    WHERE e.iguazu_partition_date BETWEEN '2026-01-21' AND '2026-03-10'
    GROUP BY 1, 2, 3
),

classified_requests AS (
    SELECT
        request_id,
        consumer_id,
        ds,
        CASE
            WHEN store_carousel_count <= 3  AND store_list_count >= 1 THEN 'broken_andromeda'
            WHEN store_carousel_count >= 10 AND store_list_count >= 1 THEN 'healthy_old_path'
        END AS path
    FROM request_path
    WHERE store_list_count >= 1
      AND (store_carousel_count <= 3 OR store_carousel_count >= 10)  -- drop ambiguous
),

-- Orders: mobile + web, same 1% consumer sample
all_orders AS (
    SELECT DATE(timestamp) AS ds, consumer_id, order_cart_id, subtotal
    FROM SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.M_CHECKOUT_PAGE_SYSTEM_CHECKOUT_SUCCESS
    WHERE DATE(timestamp) BETWEEN '2026-01-21' AND '2026-03-10'
      AND MOD(consumer_id, 100) < 1

    UNION ALL

    SELECT DATE(timestamp) AS ds, consumer_id, order_cart_id, subtotal
    FROM SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.SYSTEM_CHECKOUT_SUCCESS
    WHERE DATE(timestamp) BETWEEN '2026-01-21' AND '2026-03-10'
      AND MOD(consumer_id, 100) < 1
),

request_outcomes AS (
    SELECT
        r.path,
        COUNT(DISTINCT r.request_id)              AS total_requests,
        COUNT(DISTINCT o.order_cart_id)           AS total_orders,
        SUM(o.subtotal) / 100.0                   AS total_gmv_usd
    FROM classified_requests r
    LEFT JOIN all_orders o
        ON  r.consumer_id = o.consumer_id
        AND r.ds          = o.ds
    GROUP BY 1
)

SELECT
    path,
    total_requests,
    total_orders,
    ROUND(total_orders * 1.0 / NULLIF(total_requests, 0), 6)  AS order_rate,
    ROUND(total_gmv_usd / NULLIF(total_orders, 0),         2)  AS avg_order_value_usd,

    -- Scale counts to full population (rates need no scaling)
    total_requests * 100                                        AS est_full_population_requests,
    total_orders   * 100                                        AS est_full_population_orders,
    ROUND(total_gmv_usd * 100,                             0)  AS est_full_population_gmv_usd
FROM request_outcomes
WHERE path IS NOT NULL
ORDER BY path;
```

### Scaling to total impact

```sql
-- Once you have order_rate_broken and order_rate_healthy from above:
--
-- estimated_lost_orders =
--   (order_rate_healthy - order_rate_broken) × est_full_population_requests_broken
--
-- estimated_lost_gmv =
--   estimated_lost_orders × avg_order_value_healthy
--   + est_full_population_orders_broken × (avg_order_value_healthy - avg_order_value_broken)
```

---

## Query 5 — Date-Sampled Treatment/Control (Fast Version)

Same logic as Query 4 but optimized for speed:
- **Single day**: Feb 17 (peak breakage, ~83% broken) — strongest signal, minimal scan
- **Single hour**: 17:00 UTC (noon ET) — representative peak traffic
- **Block-level sampling**: `SAMPLE SYSTEM (1)` on the ICE table skips whole micro-partitions instead of scanning every row for a `MOD()` check

`healthy_old_path` within the same day acts as the control group — no separate baseline day needed.

| Day | Phase | Est. % Missing |
|---|---|---|
| 2026-02-17 | Phase 3 — Full treatment, peak breakage | ~83% |

```sql
WITH

-- Single day: peak broken. healthy_old_path within the same day is the control group.
sample_days AS (
    SELECT v::DATE AS ds
    FROM VALUES
        ('2026-02-17')   -- phase 3: ~83% missing (peak breakage)
    AS t(v)
),

request_path AS (
    SELECT
        e.request_id,
        e.consumer_id,
        e.iguazu_partition_date                                      AS ds,
        COUNT(CASE WHEN e.facet_type = 'store_carousel' THEN 1 END) AS store_carousel_count,
        COUNT(CASE WHEN e.facet_type = 'store_list'     THEN 1 END) AS store_list_count
    FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE AS e
        SAMPLE SYSTEM (1)                            -- block-level 1% sample (faster than MOD scan)
    JOIN sample_days s ON e.iguazu_partition_date = s.ds
    WHERE e.facet_type IN ('store_carousel', 'store_list')
      AND e.iguazu_partition_hour = 17               -- noon ET (UTC) only
    GROUP BY 1, 2, 3
),

classified_requests AS (
    SELECT
        request_id,
        consumer_id,
        ds,
        CASE
            WHEN store_carousel_count <= 3  AND store_list_count >= 1 THEN 'broken_andromeda'
            WHEN store_carousel_count >= 10 AND store_list_count >= 1 THEN 'healthy_old_path'
        END AS path
    FROM request_path
    WHERE store_list_count >= 1
      AND (store_carousel_count <= 3 OR store_carousel_count >= 10)
),

-- 1 day + 1 hour = small checkout scan; no consumer pre-filter needed
all_orders AS (
    SELECT DATE(timestamp) AS ds, consumer_id,
           COUNT(*) AS orders, SUM(subtotal) / 100.0 AS gmv_usd
    FROM SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.M_CHECKOUT_PAGE_SYSTEM_CHECKOUT_SUCCESS
    WHERE DATE(timestamp) = '2026-02-17'
      AND EXTRACT(HOUR FROM timestamp) = 17
    GROUP BY 1, 2

    UNION ALL

    SELECT DATE(timestamp) AS ds, consumer_id,
           COUNT(*) AS orders, SUM(subtotal) / 100.0 AS gmv_usd
    FROM SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.SYSTEM_CHECKOUT_SUCCESS
    WHERE DATE(timestamp) = '2026-02-17'
      AND EXTRACT(HOUR FROM timestamp) = 17
    GROUP BY 1, 2
),

request_outcomes AS (
    SELECT
        r.ds,
        r.path,
        COUNT(DISTINCT r.request_id)       AS total_requests,
        SUM(COALESCE(o.orders,   0))       AS total_orders,
        SUM(COALESCE(o.gmv_usd, 0))        AS total_gmv_usd
    FROM classified_requests r
    LEFT JOIN all_orders o
        ON  r.consumer_id = o.consumer_id
        AND r.ds          = o.ds
    WHERE r.path IS NOT NULL
    GROUP BY 1, 2
)

SELECT
    ds,
    path,
    total_requests,
    total_orders,
    ROUND(total_orders * 1.0 / NULLIF(total_requests, 0), 6)  AS order_rate,
    ROUND(total_gmv_usd / NULLIF(total_orders, 0),         2)  AS avg_order_value_usd
FROM request_outcomes
ORDER BY ds, path;
```

Reading the results: compare `order_rate` between `broken_andromeda` and `healthy_old_path`
on Feb 17. If `order_rate_broken < order_rate_healthy`, the regression has a detectable
within-day effect. If the rates are similar, the effect isn't detectable at this sample size
— widen to `SAMPLE SYSTEM (10)` or add a second day.

---

## Query 6 — CTR-Based GOV Estimation (Clicks + Impressions)

Uses store carousel CTR as a direct measure of ranking quality degradation, avoiding the
order over-counting and selection-bias-in-order-rate issues from Query 5.

**The estimation chain:**

```
lost_GOV = total_broken_impressions × (ctr_healthy − ctr_broken) × click_to_order_rate × $29.39
```

Two queries needed.

---

### Query 6a — CTR by Path (Feb 17, Noon ET)

Classifies consumers via ICE (same SYSTEM sample as Query 5), then joins to `M_CARD_VIEW`
and `M_CARD_CLICK` to compute CTR per path group. The `impressions_per_request` value
(total sample impressions ÷ classified request count) feeds into the GOV formula below.

```sql
WITH

-- Classify consumers as broken/healthy via ICE
request_path AS (
    SELECT
        e.consumer_id,
        e.request_id,
        COUNT(CASE WHEN e.facet_type = 'store_carousel' THEN 1 END) AS store_carousel_count,
        COUNT(CASE WHEN e.facet_type = 'store_list'     THEN 1 END) AS store_list_count
    FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE AS e
        SAMPLE SYSTEM (1)
    WHERE e.iguazu_partition_date = '2026-02-17'
      AND e.iguazu_partition_hour = 17
      AND e.facet_type IN ('store_carousel', 'store_list')
    GROUP BY 1, 2
),

classified_consumers AS (
    SELECT
        consumer_id,
        CASE
            WHEN AVG(store_carousel_count) <= 3  THEN 'broken_andromeda'
            WHEN AVG(store_carousel_count) >= 10 THEN 'healthy_old_path'
        END AS path
    FROM request_path
    WHERE store_list_count >= 1
    GROUP BY 1
    HAVING path IS NOT NULL
),

-- Homepage store carousel impressions for classified consumers only
impressions AS (
    SELECT
        c.path,
        v.consumer_id,
        v.store_id,
        v.carousel_name,
        v.card_position,
        v.vertical_position
    FROM IGUAZU.CONSUMER.M_CARD_VIEW v
    INNER JOIN classified_consumers c ON v.consumer_id = c.consumer_id
    WHERE DATE(v.timestamp)                = '2026-02-17'
      AND EXTRACT(HOUR FROM v.timestamp)   = 17
      AND v.container = 'cluster'
      AND v.page      = 'explore_page'
      AND v.store_id  IS NOT NULL
),

-- Left-join clicks onto impressions using composite key from data sources doc
impression_clicks AS (
    SELECT
        i.path,
        CASE WHEN k.consumer_id IS NOT NULL THEN 1 ELSE 0 END AS clicked
    FROM impressions i
    LEFT JOIN IGUAZU.CONSUMER.M_CARD_CLICK k
        ON  i.consumer_id       = k.consumer_id
        AND i.store_id          = k.store_id
        AND i.carousel_name     = k.carousel_name
        AND i.card_position     = k.card_position
        AND i.vertical_position = k.vertical_position
        AND DATE(k.timestamp)              = '2026-02-17'
        AND EXTRACT(HOUR FROM k.timestamp) = 17
        AND k.container = 'cluster'
        AND k.page      = 'explore_page'
)

SELECT
    path,
    COUNT(*)                                              AS impressions,
    SUM(clicked)                                          AS clicks,
    ROUND(SUM(clicked) * 1.0 / NULLIF(COUNT(*), 0), 6)   AS ctr
FROM impression_clicks
GROUP BY 1
ORDER BY 1;
```

---

### Query 6b — Click-to-Order Rate (Baseline, Jan 15–19)

Uses consumer-day granularity — "of consumers who clicked a carousel, what fraction ordered
that day?" — to avoid the over-counting problem from the impression-level join.

```sql
WITH

clickers AS (
    SELECT DISTINCT
        DATE(timestamp) AS ds,
        consumer_id
    FROM IGUAZU.CONSUMER.M_CARD_CLICK
    WHERE DATE(timestamp) BETWEEN '2026-01-15' AND '2026-01-19'
      AND container = 'cluster'
      AND page      = 'explore_page'
      AND store_id  IS NOT NULL
),

orders AS (
    SELECT DATE(timestamp) AS ds, consumer_id
    FROM SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.M_CHECKOUT_PAGE_SYSTEM_CHECKOUT_SUCCESS
    WHERE DATE(timestamp) BETWEEN '2026-01-15' AND '2026-01-19'
    UNION ALL
    SELECT DATE(timestamp) AS ds, consumer_id
    FROM SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.SYSTEM_CHECKOUT_SUCCESS
    WHERE DATE(timestamp) BETWEEN '2026-01-15' AND '2026-01-19'
)

SELECT
    COUNT(*)                                                   AS unique_clicker_days,
    COUNT(DISTINCT o.consumer_id || o.ds)                      AS clicker_days_with_order,
    ROUND(COUNT(DISTINCT o.consumer_id || o.ds) * 1.0
          / NULLIF(COUNT(*), 0), 6)                            AS click_to_order_rate
FROM clickers c
LEFT JOIN orders o ON c.consumer_id = o.consumer_id AND c.ds = o.ds;
```

---

### GOV Impact Formula

Apply after both queries return:

```
-- From Query 6a:
ctr_delta               = ctr_healthy − ctr_broken
impressions_per_request = total_impressions_in_sample / classified_requests_in_sample
                          (use broken_andromeda impressions / broken_andromeda request count
                           from classified_consumers as denominator)

-- Scale to full regression period:
total_broken_requests   = 2,987,633,468 × 0.63         -- appendix A2 avg broken fraction
total_broken_impressions = total_broken_requests × impressions_per_request

-- Chain:
lost_clicks  = total_broken_impressions × ctr_delta
lost_orders  = lost_clicks × click_to_order_rate        -- from Query 6b
lost_GOV     = lost_orders × $29.39                     -- baseline avg order value
```

**Caveat:** Selection bias still present — Andromeda users may click differently from
non-Andromeda users independent of ranking quality. The `ctr_delta` is therefore an upper
bound on the regression-attributable CTR loss, not a clean causal estimate.

---

## Query 7 — Before/After CTR (Android, Same Day of Week)

The cleanest available comparison. No broken/healthy classification, no ICE table, no
selection bias from DV assignment.

**Why Android only:**

| Day | Android DV state | iOS DV state |
|---|---|---|
| Jan 20, 2026 (pre-bug) | 100% Andromeda, **code healthy** (regression introduced Jan 21) | 0% Andromeda (iOS not rolled until Jan 28–29) |
| Feb 17, 2026 (peak bug) | 100% Andromeda, **code broken** (~83% events missing) | 100% Andromeda, broken |

Android is the same population on both days (100% Andromeda), with the only variable being
whether the logging bug was present. iOS is a completely different population across the two
days — excluded.

**Why Jan 20:**
Same day of week as Feb 17 (both Tuesday). One day before the regression was introduced.
Same season as Jan 21 regression start — no 4-week seasonality gap.

**`container = 'cluster'`** targets carousel containers specifically. On the explore page
there are also store_list (All Stores) containers with different CTR patterns. Keep this
filter to isolate carousel engagement — the part of the product affected by the regression.

**Sampling:** ICE uses `SAMPLE SYSTEM (1)`. M_CARD_VIEW and M_CARD_CLICK are inner-joined
to ICE-classified consumers, so they are automatically scoped to the ICE sample — no
separate MOD filter needed. `SAMPLE SYSTEM` is not applied to M_CARD_VIEW/M_CARD_CLICK
because they are joined to each other on `consumer_id` (SYSTEM picks different blocks per
table, breaking the join).

**Schema note:** `iguazu_partition_hour` does not exist on `M_CARD_VIEW` / `M_CARD_CLICK`.
Use `EXTRACT(HOUR FROM timestamp)` instead.

**Output groups:**

| group | day | meaning |
|---|---|---|
| `pre_bug` | Jan 20 | All consumers — no bug yet, all healthy |
| `peak_bug_healthy` | Feb 17 | Store carousel events present (≥10) — old code path |
| `peak_bug_broken` | Feb 17 | Store carousel events absent/minimal (≤3) — broken path |

`peak_bug_healthy` vs `peak_bug_broken` removes seasonality (same day, same hour).
`pre_bug` vs `peak_bug_broken` gives the full before/after on the affected population type.

```sql
WITH

-- Step 1: Classify consumers via ICE
-- Jan 20: pre-bug — all consumers expected healthy (store_carousel present)
-- Feb 17: split broken (≤3 store_carousel) vs healthy (≥10)
ice_requests AS (
    SELECT
        e.consumer_id,
        e.iguazu_partition_date                                       AS ds,
        COUNT(CASE WHEN e.facet_type = 'store_carousel' THEN 1 END)  AS store_carousel_count,
        COUNT(CASE WHEN e.facet_type = 'store_list'     THEN 1 END)  AS store_list_count
    FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE AS e
        SAMPLE SYSTEM (1)
    WHERE e.iguazu_partition_date IN ('2026-01-20', '2026-02-17')
      AND e.iguazu_partition_hour  = 17
      AND e.facet_type IN ('store_carousel', 'store_list')
    GROUP BY 1, 2
),

classified_consumers AS (
    SELECT
        consumer_id,
        ds,
        CASE
            WHEN ds = '2026-01-20'                                    THEN 'pre_bug'
            WHEN ds = '2026-02-17' AND store_carousel_count >= 10     THEN 'peak_bug_healthy'
            WHEN ds = '2026-02-17' AND store_carousel_count <= 3
                                   AND store_list_count    >= 1       THEN 'peak_bug_broken'
        END AS group_label
    FROM ice_requests
    WHERE store_list_count >= 1
    HAVING group_label IS NOT NULL
),

-- Step 2: Carousel impressions for ICE-classified consumers only
-- No MOD filter needed — inner join to classified_consumers scopes to ICE sample
views AS (
    SELECT
        c.group_label,
        v.consumer_id,
        v.ds,
        v.store_id,
        v.carousel_name,
        v.card_position,
        v.vertical_position
    FROM (
        SELECT
            DATE(timestamp)                AS ds,
            consumer_id,
            store_id,
            carousel_name,
            card_position,
            vertical_position
        FROM IGUAZU.CONSUMER.M_CARD_VIEW
        WHERE DATE(timestamp)              IN ('2026-01-20', '2026-02-17')
          AND EXTRACT(HOUR FROM timestamp)  = 17
          AND LOWER(platform)               = 'android'
          AND container                     = 'cluster'   -- carousel containers only; not store_list
          AND page                          = 'explore_page'
          AND store_id                      IS NOT NULL
    ) v
    INNER JOIN classified_consumers c
        ON  v.consumer_id = c.consumer_id
        AND v.ds          = c.ds
),

-- Step 3: Clicks — pre-filter to same dates/hour/platform/filters; join to views handles scoping
clicks AS (
    SELECT DISTINCT
        DATE(timestamp)                AS ds,
        consumer_id,
        store_id,
        carousel_name,
        card_position,
        vertical_position
    FROM IGUAZU.CONSUMER.M_CARD_CLICK
    WHERE DATE(timestamp)              IN ('2026-01-20', '2026-02-17')
      AND EXTRACT(HOUR FROM timestamp)  = 17
      AND LOWER(platform)               = 'android'
      AND container                     = 'cluster'
      AND page                          = 'explore_page'
      AND store_id                      IS NOT NULL
)

SELECT
    v.ds,
    v.group_label,
    COUNT(*)                                                     AS impressions,
    COUNT(k.consumer_id)                                         AS clicks,
    ROUND(COUNT(k.consumer_id) * 1.0 / NULLIF(COUNT(*), 0), 6)  AS ctr
FROM views v
LEFT JOIN clicks k
    ON  v.ds                = k.ds
    AND v.consumer_id       = k.consumer_id
    AND v.store_id          = k.store_id
    AND v.carousel_name     = k.carousel_name
    AND v.card_position     = k.card_position
    AND v.vertical_position = k.vertical_position
GROUP BY 1, 2
ORDER BY 1, 2;
```

---

### Query 7 — GOV Impact Formula

Two independent CTR deltas to compute:

```
-- 1. Within-day causal estimate (controls for seasonality):
ctr_delta_within_day  = ctr_peak_bug_healthy − ctr_peak_bug_broken

-- 2. Before/after estimate on broken population (has 4-week seasonality gap):
ctr_delta_before_after = ctr_pre_bug − ctr_peak_bug_broken

-- Scale broken impressions to full Android population:
-- ICE used SAMPLE SYSTEM (1) → scale factor ≈ 100×
total_broken_impressions_1hr = peak_bug_broken_impressions × 100

-- Scale to full regression period (Android 100% broken Jan 21–Mar 10, 49 days):
-- Impressions measured at 1 hour; rough scale to full day ~16 peak-adjacent hours:
total_broken_impressions_regression = total_broken_impressions_1hr × 16 × 49

-- Apply chosen ctr_delta (use within-day as primary, before/after as check):
lost_clicks  = total_broken_impressions_regression × ctr_delta
lost_orders  = lost_clicks × click_to_order_rate    -- from Query 6b
lost_GOV     = lost_orders × $29.39
```

**Reading the results:**
- If `ctr_peak_bug_broken < ctr_peak_bug_healthy`: within-day ranking degradation detected
- If `ctr_peak_bug_broken < ctr_pre_bug`: before/after degradation detected
- If `ctr_peak_bug_broken > ctr_pre_bug` but `< ctr_peak_bug_healthy`: seasonality inflating
  Feb 17 baseline, but within-day comparison still valid

---

## Limitations

| Issue                        | Note                                                                                                                                                                                                                                                                                                                                                                                              |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Logging ≠ product bug        | Carousels were still served; impact flows through ML model degradation, which is gradual and hard to isolate cleanly                                                                                                                                                                                                                                                                              |
| Seasonality                  | Feb/Mar have different demand patterns than Jan; before/after comparisons are confounded                                                                                                                                                                                                                                                                                                          |
| Attribution                  | Daily request:order ratio assumes stable attribution; no direct session-to-order join                                                                                                                                                                                                                                                                                                             |
| Mobile + web                 | Queries 2–4 union both platforms; verify field names are consistent across the two tables                                                                                                                                                                                                                                                                                                         |
| Broken path proxy            | Query 4 uses `store_carousel_count <= 3` as a proxy for `isAndromedaStoreCarouselEnabled = true`. The broken path averaged 1.85 store_carousel events/request (max ~3); the healthy path averaged 13.54. Requests with 4–9 events are excluded as ambiguous. Validate by checking that the broken/healthy split matches the known DV ramp schedule (e.g., ~0% broken on Jan 21, ~100% by Feb 12). |
| Session-to-order attribution | Query 4 joins on `consumer_id + date` — one consumer can have multiple requests per day. A tighter join using `dd_session_id` on the checkout table would reduce noise if that field is available.                                                                                                                                                                                                |
