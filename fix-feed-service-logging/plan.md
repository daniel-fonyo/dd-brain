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

## Facet Matching Coverage (Frank's Analysis — PR #61377)

Frank ran debug logging on sandbox traffic to verify `addXVerticalCategorySectionEvents` matches all loggable facets. Results: **97.6–98.8% coverage** across two samples. The only unmatched loggable facet is banner carousel (separate pipeline). All store carousels, item carousels, collections, feed placements, reels, tiles, store rows, and CC store rows are matched.

This confirms: no other carousel types are silently dropped by the `when` block — the Jan 21 regression was isolated to the `store_carousel:` infix bug.

## Next Steps: Deeper Gap Analysis

Frank proposed two layers of further validation beyond facet matching (see `context/gap-analysis-plan.md`):

1. **Shadow traffic validation** — run debug logging on 1% prod traffic to confirm facet matching at scale (sandbox only covers limited address diversity)
2. **Event field completeness** — even when a facet matches and `generateEvents()` runs, fields may be missing or incorrect (e.g. store carousels' `page_index` and `page_size` are known missing). This requires code-level audit of each `ContainerEventsGenerator` subclass, not runtime validation.

## Open Questions

1. Query 7 (Android before/after CTR) — last remaining impact analysis with clean comparison design
2. ML model degradation framing — quantify data quality loss and identify which models trained on the affected window
3. Backfill decision — which ML pipelines consumed store carousel data during Jan 21–Mar 10?
4. Priority and LoE for gap analysis work — to be discussed at post-mortem
