# Impact Analysis Segments
## Stable DV Windows for GOV Estimation

*Context: An analyst needs exact traffic and dates impacted to estimate GOV impact from the store carousel Iguazu event regression (Jan 21 - Mar 10, 2026). Android was 100% on Andromeda DV throughout the bug, making it the cleanest single-platform estimate. This document identifies additional segments with stable DV states (1+ weeks) for cross-validation.*

*Source: DV revision history in [regression doc](./Store%20Carousel%20Iguazu%20Events%20Regression.md) Appendix A4, daily breakdown in A2.*

---

## DV Structure Recap

The `andromeda_homepage_store_carousel` DV has multiple segments. A request is routed to one segment; the segment's rollout % determines whether Andromeda (the broken code path) is enabled.

| Segment | State pre-bug (Jan 20) | Bug introduced Jan 21 |
|---|---|---|
| Android Users | 100% since Dec 19 | 100% broken from Jan 21 |
| iOS Users | 0% until Jan 28 | Ramped 0%→100% Jan 28-29 |
| iOS hitch fix w/ Holdout | 25% since Dec 19 | 25%→50%→100% Feb 9-10; holdout removed Feb 12 |
| hide_delivery_fee | N/A | Added Feb 26, rolled 0%→100%→0%, removed Mar 2 |

---

## Stable Segments (1+ Week, Consistent DV State)

### 1. Android Pre-Bug Baseline (Jan 15-20)
- **DV state**: 100% Andromeda, code **healthy** (bug introduced Jan 21)
- **Duration**: 6 days
- **Use**: Baseline for Android before/after comparison

| Date | Requests | Events | % Missing |
|---|---|---|---|
| Jan 15-19 | ~366M (5 days, from impact-report.md) | Normal | 0% |
| Jan 20 | ~68M (est. from Jan 21 volume) | Normal | 0% |

### 2. Android Full Bug Period (Jan 21 - Mar 10)
- **DV state**: 100% Andromeda, code **broken**
- **Duration**: 49 days
- **Use**: Primary treatment period; already being analyzed by analyst

Daily traffic from A2 (total, not Android-only — needs platform filter for Android-specific analysis):

| Date | Requests | Actual Events | Est. Lost | % Missing |
|---|---|---|---|---|
| Jan 21 | 68,456,206 | 1,507,862,041 | 125,347,433 | 7.67% |
| Jan 22 | 63,273,961 | 1,445,475,606 | 64,097,282 | 4.25% |
| Jan 23 | 72,798,030 | 1,516,043,254 | 220,752,272 | 12.71% |
| Jan 24 | 75,138,199 | 1,545,014,271 | 247,612,368 | 13.81% |
| Jan 25 | 65,734,842 | 1,189,127,619 | 379,156,296 | 24.18% |
| Jan 26 | 63,527,659 | 1,106,744,569 | 408,880,976 | 26.98% |
| Jan 27 | 71,647,966 | 1,355,581,929 | 353,775,684 | 20.70% |

*Note: A2 numbers are all-platform. The analyst will need to filter by platform in Snowflake to get Android-only and iOS-only request counts. The `platform` field on `CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE` or client-side events (`M_CARD_VIEW`) can be used.*

### 3. iOS Minimal Exposure (Jan 21-27) — "Mostly Healthy" Control
- **DV state**: iOS Users at 0%, iOS hitch fix at 25%
- **Duration**: 7 days
- **Estimated % iOS traffic affected**: Small. Only the "iOS hitch fix" segment (unknown % of total iOS traffic) was at 25% treatment. The vast majority of iOS traffic was NOT on the Andromeda path and therefore NOT affected by the bug.
- **Use**: iOS baseline period — most iOS users saw healthy store carousel rankings during this window.
- **% Missing from A2**: 4-27% (all-platform; iOS-only would be much lower)

### 4. iOS Full Exposure — Phase 3 (Feb 12-25)
- **DV state**: All iOS segments at 100% (Rev 106 removed holdout Feb 12)
- **Duration**: 14 days
- **Use**: iOS treatment period. All iOS traffic routed through broken path.
- **Daily data from A2 (all-platform)**:

| Date | Requests | % Missing |
|---|---|---|
| Feb 12 | 59,342,064 | 52.54% |
| Feb 13 | 51,488,878 | 79.36% |
| Feb 14 | 60,635,773 | 80.76% |
| Feb 15 | 71,902,867 | 83.37% |
| Feb 16 | 62,938,335 | 83.53% |
| Feb 17 | 53,413,329 | 82.56% |
| Feb 18 | 45,708,076 | 81.44% |
| Feb 19 | 56,944,223 | 76.93% |
| Feb 20 | 55,955,848 | 82.17% |
| Feb 21 | 59,576,173 | 82.55% |
| Feb 22 | 60,550,048 | 83.64% |
| Feb 23 | 47,959,331 | 82.58% |
| Feb 24 | 50,898,276 | 82.69% |
| Feb 25 | 51,576,345 | 82.10% |

### 5. iOS Full Exposure — Phase 5 (Mar 3-10)
- **DV state**: All segments at 100%, hide_delivery_fee removed (Rev 114, Mar 2)
- **Duration**: 8 days
- **Use**: Second iOS treatment window; cross-validates Phase 3.
- **Daily data from A2 (all-platform)**:

| Date | Requests | % Missing |
|---|---|---|
| Mar 3 | 50,879,296 | 81.02% |
| Mar 4 | 46,161,433 | 78.78% |
| Mar 5 | 59,827,115 | 70.35% |
| Mar 6 | 51,722,553 | 78.65% |
| Mar 7 | 55,680,734 | 78.08% |
| Mar 8 | 55,099,132 | 78.50% |
| Mar 9 | 45,479,414 | 78.73% |
| Mar 10 | 43,491,726 | 78.05% |

### Segments to Skip (Unstable DV)
| Period | Why skip |
|---|---|
| Jan 28-29 | iOS Users ramping 0%→100% in 2 days |
| Feb 9-11 | iOS hitch fix ramping 25%→100% |
| Feb 26 - Mar 2 | hide_delivery_fee segment added, ramped 100%, rolled back, removed |

---

## Recommended Analysis Approaches

### A. iOS Within-Platform Before/After (Primary iOS Estimate)
**Compare**: iOS Jan 21-27 (mostly healthy) vs iOS Feb 12-25 (fully broken)

| Advantage | Limitation |
|---|---|
| Same platform — no Android vs iOS population bias | ~2 week seasonality gap (but smaller than Android's 4-week gap) |
| 7-day control vs 14-day treatment — both >1 week | Jan 21-27 is not perfectly clean — ~25% of hitch fix segment was on Andromeda |
| Neither window includes Super Bowl (Feb 9) or Valentine's Day (Feb 14... but Feb 14 is in treatment) | Valentine's Day falls in the treatment window — could inflate treatment metrics |

**Approach**: Filter ICE table and checkout tables to `platform = 'ios'`. Compare order rate and GOV/request between the two windows. The causal chain (logging → ML retraining → ranking degradation) means early bug period (Jan 21-27) shouldn't show impact from the bug itself — ML models hadn't retrained yet. So this comparison also captures the *accumulation* of ML degradation over time.

### B. Cross-Platform Same-Period (Seasonality-Free)
**Compare**: Android Jan 21-27 (100% broken) vs iOS Jan 21-27 (~0% broken)

| Advantage | Limitation |
|---|---|
| Same 7-day window — eliminates seasonality entirely | Android vs iOS Cx differ in characteristics (as analyst noted) |
| Clear treatment/control: Android fully broken, iOS mostly healthy | ML models hadn't retrained yet in week 1 — minimal expected impact |

**Use**: This is a negative control. If metrics are similar (expected, since ML hasn't retrained), it confirms the causal mechanism is gradual. If metrics differ, it reveals a direct impact channel we hadn't considered.

### C. iOS Dose-Response Across Phases
**Compare**: iOS metrics across 3 stable phases:
1. Jan 21-27: ~0% exposure
2. Feb 12-25: ~100% exposure (14 days into full iOS rollout)
3. Mar 3-10: ~100% exposure (6+ weeks into full iOS rollout)

If the bug had CX impact through ML degradation, metrics should progressively degrade from phase 1 → 2 → 3 as models retrained on increasingly bad data.

### D. Android Narrow Pre/Post (Negative Control)
**Compare**: Android Jan 15-20 (healthy) vs Android Jan 21-27 (broken, week 1 only)

Expected result: no measurable difference, because ML models haven't retrained yet in week 1. Useful to confirm the causal chain is logging → ML → ranking, not a direct product impact. If there IS a difference in week 1, that changes the impact narrative entirely.

---

## What to Provide the Analyst

The analyst needs:
1. **Platform-filtered daily request counts** from ICE for Android and iOS separately (A2 is all-platform)
2. **The DV state table above** — maps each date to exact % affected per platform
3. **The segment windows** — which date ranges have stable DV state for >1 week
4. **The recommended comparisons** — prioritized by statistical cleanliness

### SQL to Get Platform-Filtered Daily Counts

```sql
SELECT
    iguazu_partition_date AS ds,
    CASE
        WHEN LOWER(platform) = 'android' THEN 'android'
        WHEN LOWER(platform) = 'ios'     THEN 'ios'
        ELSE 'other'
    END AS platform_group,
    APPROX_COUNT_DISTINCT(request_id) AS requests,
    COUNT(CASE WHEN facet_type = 'store_carousel'
               AND horizontal_position_in_facet = 0 THEN 1 END) AS store_carousel_events
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
WHERE iguazu_partition_date BETWEEN '2026-01-15' AND '2026-03-11'
GROUP BY 1, 2
ORDER BY 1, 2;
```

*Note: filter by `iguazu_partition_hour` if needed to reduce scan cost. The `platform` column on ICE should match the requesting client's platform.*
