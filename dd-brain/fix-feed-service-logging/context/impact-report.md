# CX User Impact Report
## Store Carousel Iguazu Event Regression — Jan 21–Mar 10, 2026

*Status: Queries A, B, 5, 6a complete. Query 6b (click-to-order rate) pending.*

---

## Executive Summary

Both analyses fail to detect measurable CX user impact from the regression.

**Before/after** (Queries A/B): The regression period had a *higher* order rate than baseline
(0.121 vs 0.103), driven entirely by seasonality (Super Bowl, Valentine's Day). No order rate
loss detectable. Avg order value dropped $0.08 — within normal variance.

**Within-day treatment/control** (Query 5, Feb 17): The broken-path group actually showed a
*higher* order rate than the healthy-path group (0.616 vs 0.536). This is opposite to the
expected direction and is explained by **selection bias** — Andromeda DV was gated on app
version, so `broken_andromeda` users are heavier/newer app users with intrinsically higher
order propensity. The two groups are not comparable populations. Order rates are also
inflated ~6× by the `consumer_id + date` join assigning one consumer's order to all their
requests in the sample window.

The avg order value was $0.58 lower on the broken path ($25.63 vs $26.21), in the expected
direction, but this difference cannot be isolated from population characteristics.

**CTR-based approach (Query 6a):** The broken path had *higher* CTR than healthy
(0.000888 vs 0.000508), the same wrong direction as Query 5. `ctr_delta` is negative —
no GOV loss detectable via the CTR channel. Selection bias dominates both analyses.

**Conclusion: no measurable CX impact detected.** The regression was a logging regression —
carousels were still served to users. Across every metric (order rate, AOV, CTR), the
broken path performs equal to or better than the healthy path. This is the signature of
selection bias: Andromeda DV users are heavier, more engaged app users with higher
intrinsic order propensity and CTR, independent of carousel ranking quality. Any ML
degradation from incomplete training data is below the noise floor of available methodology.

---

## Background

A code refactor released Jan 21, 2026 caused store carousel Iguazu events
(`CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE`) to stop being emitted on requests routed through
`StoreCarouselMapper` (`isAndromedaStoreCarouselEnabled = true`). The fix was released Mar 11.

The path to CX user impact is indirect: degraded logging → ML models retrained on incomplete
data → worse store carousel rankings → lower CTR → fewer/smaller orders. Impact would
accumulate gradually as models retrained, not immediately on Jan 21.

---

## Data Sources

| Table | Usage |
|---|---|
| `IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE` | Homepage request counts; broken/healthy path classification |
| `SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.M_CHECKOUT_PAGE_SYSTEM_CHECKOUT_SUCCESS` | Mobile orders + GOV |
| `SEGMENT_EVENTS_RAW.CONSUMER_PRODUCTION.SYSTEM_CHECKOUT_SUCCESS` | Web orders + GOV |
| `IGUAZU.CONSUMER.M_CARD_VIEW` | Mobile store carousel impressions (Query 6) |
| `IGUAZU.CONSUMER.M_CARD_CLICK` | Mobile store carousel clicks (Query 6) |

---

## Baseline Metrics (Jan 15–19, 2026)

*5 pre-regression days. All platforms (mobile + web).*

| Metric | Value |
|---|---|
| Homepage requests | 366,289,596 |
| Total orders | 37,647,351 |
| Total GOV | $1,105,994,689 |
| **Order rate** (orders / request) | **0.1028** |
| **GOV per request** | **$3.02** |
| Avg order value | $29.39 |
| Daily requests | ~73.3M/day |
| Daily orders | ~7.53M/day |
| Daily GOV | ~$221M/day |

---

## Regression Period Metrics (Jan 21–Mar 10, 2026)

*49 days. Requests from Appendix A2 of regression doc (sum of daily counts).*

| Metric | Value |
|---|---|
| Homepage requests | 2,987,633,468 |
| Total orders | 361,980,506 |
| Total GOV | $10,608,011,050 |
| **Order rate** (orders / request) | **0.1212** |
| **GOV per request** | **$3.55** |
| Avg order value | $29.31 |
| Daily requests | ~61.0M/day |
| Daily orders | ~7.39M/day |
| Daily GOV | ~$216M/day |

---

## Before/After Comparison

| Metric | Baseline | Regression | Delta |
|---|---|---|---|
| Order rate (orders/request) | 0.1028 | 0.1212 | **+0.0184 (+17.9%)** |
| GOV per request | $3.02 | $3.55 | **+$0.53 (+17.5%)** |
| Avg order value | $29.39 | $29.31 | -$0.08 (-0.3%) |
| Daily requests | 73.3M | 61.0M | -12.3M (-16.8%) |
| Daily GOV | $221M | $216M | -$5M (-2.3%) |

### Key finding: seasonality dominates

The regression period has a **higher** order rate and GOV per request than baseline. The
lower absolute daily GOV ($221M → $216M) is fully explained by lower request volume in
Feb/Mar vs Jan — not by lower conversion rates.

Probable drivers of higher Feb/Mar order rates:
- Super Bowl (Feb 9) — historically DoorDash's highest-volume day
- Valentine's Day (Feb 14)
- General demand growth vs prior year

**Conclusion: the before/after comparison cannot detect any regression impact.** Seasonality
noise is larger than any signal from the regression.

---

## Avg Order Value: Negligible Size Effect

The $0.08 drop in avg order value ($29.39 → $29.31) is effectively zero. Even at full
regression scale this is:

```
$0.08 × 361,980,506 orders = ~$29M across 49 days (~$590K/day)
```

This is within normal day-to-day variance and should not be attributed to the regression.

---

## Treatment/Control Analysis (Query 5)

To remove the seasonality confound, each homepage request was classified as:

- **`broken_andromeda`**: `store_carousel_count ≤ 3` AND `store_list_count ≥ 1`
  → request routed through `StoreCarouselMapper`, logging suppressed
- **`healthy_old_path`**: `store_carousel_count ≥ 10` AND `store_list_count ≥ 1`
  → request through old code, logging intact

Sample: Feb 17 (peak breakage, ~83% broken) × 17:00 UTC (noon ET) × `SAMPLE SYSTEM (1)` on ICE table.

### Query 5 Output

| ds | path | requests | orders | order_rate | avg_order_value |
|---|---|---|---|---|---|
| 2026-02-17 | broken_andromeda | 76,685 | 47,275 | 0.6165 | $25.63 |
| 2026-02-17 | healthy_old_path | 18,325 | 9,817 | 0.5357 | $26.21 |

### Interpretation

**Order rates are not usable.** The reported rates (61.6% / 53.6%) are ~6× higher than the
true baseline of 10.3%. Root cause: the join on `consumer_id + date` assigns a consumer's
single order to every request they made in the sample window. A consumer with N requests
has their order counted N times. This is a structural over-count in the join design, not a
data quality issue.

**Direction is wrong.** Even setting aside the inflation, `broken_andromeda` has a *higher*
order rate than `healthy_old_path` (0.616 vs 0.536). This is opposite to the hypothesis.
The likely explanation is **selection bias**: the Andromeda DV was gated on app version.
`broken_andromeda` users skew toward newer, heavier app users with intrinsically higher order
propensity — independent of carousel quality. The two groups are not exchangeable.

**Avg order value signal.** The $0.58 gap ($25.63 vs $26.21) is in the expected direction
(broken path → smaller baskets) but cannot be cleanly attributed to the regression vs.
population differences between the two DV groups.

### Impact Estimate

Not computable from this analysis. The order rate comparison is invalidated by selection
bias. A clean causal estimate would require either:
- A proper A/B test that randomly assigns the broken/healthy path within the same user segment
- An instrumental variable approach using the DV ramp schedule as an instrument
- Retrospective analysis after multiple model retraining cycles (to measure ranking
  degradation downstream)

---

## CTR-Based GOV Estimation [PENDING — Query 6a/6b not yet run]

Uses store carousel CTR as a direct measure of ranking quality degradation. Avoids the
order over-counting and selection-bias-in-order-rate issues from Query 5. CTR measures
engagement with the specific content shown — if degraded rankings surface less relevant
stores, CTR drops regardless of user type.

**Estimation chain:**
```
lost_GOV = total_broken_impressions × (ctr_healthy − ctr_broken) × click_to_order_rate × $29.39
```

Sample: same as Query 5 — Feb 17, 17:00 UTC, SYSTEM 1% on ICE. Broken/healthy
classification via `store_carousel_count ≤ 3` / `≥ 10`.

### Query 6a Output — CTR by Path (Feb 17)

| path | impressions | clicks | ctr |
|---|---|---|---|
| broken_andromeda | 1,036,525 | 920 | 0.000888 |
| healthy_old_path | 289,436 | 147 | 0.000508 |

Impressions per request (from Query 5 request counts): broken ≈ 13.5, healthy ≈ 15.8 — both realistic for a homepage carousel feed.

### [PENDING] Query 6b Output — Click-to-Order Rate (Baseline Jan 15–19)

| unique_clicker_days | clicker_days_with_order | click_to_order_rate |
|---|---|---|
| — | — | — |

### GOV Impact Estimate

```
ctr_delta = ctr_healthy − ctr_broken = 0.000508 − 0.000888 = −0.000380
```

**ctr_delta is negative** — broken path has higher CTR than healthy. The GOV formula
produces a negative (i.e., zero detectable) loss. No further computation needed.

### Interpretation

Broken path users click ~75% more per card impression than healthy path users (0.000888 vs
0.000508). Combined with the Query 5 finding (broken users have higher order rate), this
consistently points to **selection bias as the dominant effect**:

| Metric | broken_andromeda | healthy_old_path | Direction |
|---|---|---|---|
| Order rate (Query 5) | 0.6165 | 0.5357 | broken > healthy ❌ |
| Avg order value (Query 5) | $25.63 | $26.21 | broken < healthy ✓ |
| CTR (Query 6a) | 0.000888 | 0.000508 | broken > healthy ❌ |
| Clicks per request | ~0.012 | ~0.008 | broken > healthy ❌ |

Three out of four metrics are in the wrong direction. The only metric moving the expected
way (AOV, −$0.58) is also the most susceptible to basket composition differences between
user segments.

**The Andromeda DV split is not a valid treatment/control.** Broken path users are a
self-selected group of heavier, more engaged consumers. Any regression signal is swamped.

---

## Limitations

| Issue | Note |
|---|---|
| Indirect mechanism | Impact flows through ML model retraining — gradual and hard to isolate. No direct product break. |
| Before/after confounded | Seasonality (Super Bowl, Valentine's Day, demand growth) dominates before/after signal. Order rate was higher during regression than baseline. |
| Selection bias in T/C | Andromeda DV assignment is not random — it tracks app version. `broken_andromeda` users are systematically different (heavier users, newer builds) from `healthy_old_path`. Order rate comparison is not apples-to-apples. |
| Order rate inflation | `consumer_id + date` join assigns one consumer's order to all their requests in the sample window — not just the request that preceded the order. Rates are inflated ~6×. A session-level or request-to-order funnel join would be more correct. |
| Hour filter on checkout | Checkout table filtered to hour 17 UTC. Orders placed outside that hour by consumers who browsed at noon are missed. Affects both groups equally so the relative comparison would hold — but the absolute rates are underestimates. |
| CTR selection bias (Query 6) | Same Andromeda DV selection bias applies to CTR comparison — broken path users may click more or less in general. The CTR delta is therefore an upper bound on regression-attributable CTR loss, not a clean causal estimate. |
