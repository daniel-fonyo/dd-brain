# Investigation: Store Carousel Iguazu Events Regression

## Summary

A refactor (PR #56553, released Jan 21, 2026) broke store carousel Iguazu event emission for 49 days. ~32B events lost. Fixed Mar 11 via PR #60973.

## Root Cause

`StoreCarouselMapper` omitted the `facetIdPrefix` parameter when calling the shared builder, producing facet IDs like `carousel.standard:favorites` instead of `carousel.standard:store_carousel:favorites`. The `IguazuEventUtil` event generator matches facet IDs by string prefix to route to the correct generator — unrecognized IDs silently fell through with no error/metric/log.

## Impact Analysis Status

| Analysis | Status | Finding |
|---|---|---|
| Regression doc (what/why/how) | Done | ~32B events lost, ~45% average missing rate |
| Before/after order rate | Done | No signal — seasonality dominates (+17.9% order rate during regression) |
| Treatment/control (broken vs healthy path) | Done | Selection bias invalidates — broken path shows *higher* order rate and CTR |
| CTR-based GOV estimation (Query 6a) | Done | ctr_delta negative (broken > healthy) — no detectable loss |
| Before/after Android CTR (Query 7) | Pending | Cleanest design — same platform, same DV, only variable is the bug |

## Key Finding

No measurable CX user impact detected through any behavioral analysis. Selection bias from the Andromeda DV (gated on app version) swamps any regression signal — broken-path users are systematically heavier/more engaged consumers.

## Open Questions

1. Query 7 (Android before/after CTR) — last remaining analysis with clean comparison design
2. ML model degradation framing — quantify data quality loss and identify which models trained on the affected window, rather than trying to attribute GOV directly
3. Backfill decision — which ML pipelines consumed store carousel data during Jan 21–Mar 10?
