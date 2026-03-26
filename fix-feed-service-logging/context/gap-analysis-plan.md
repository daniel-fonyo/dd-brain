# Iguazu Event Gap Analysis Plan

Source: Frank Zhang's PR #61377 and Slack discussion (Mar 26, 2026).

## Background

PR #61377 added debug logging to `IguazuEventUtil.addXVerticalCategorySectionEvents` to verify which facet types in the homepage `bodyList` are matched (events generated) vs unmatched (silently skipped). Sandbox analysis showed 97.6–98.8% coverage of loggable content. The store carousel regression is confirmed fixed. No other carousel types are being silently dropped.

But facet matching is only layer 1 of 3. Two deeper layers remain unvalidated.

## Three Layers of Validation

### Layer 1: Facet ID Matching (DONE — PR #61377)

**Question**: Does every loggable facet in `bodyList` hit a `when` branch?

**Method**: Debug logging (`[fz-debug] match/no-match`) on sandbox traffic.

**Result**: Yes. 81/83 and 80/81 loggable facets matched across two samples. Only banner carousel (separate pipeline) and an empty `list-component-wrapper` were unmatched.

**Limitation**: Sandbox traffic covers limited address diversity. A 1% shadow on prod would confirm at scale.

### Layer 2: Event Generation Success (NOT YET VALIDATED)

**Question**: When a facet matches and `generateEvents(retrievalData, layoutData)` is called, does it actually produce and send Iguazu events?

**Why it might fail silently**:
- `generateEvents()` could return an empty list if internal guards filter out the content (e.g. `storeId > 0` guard, empty children)
- Exceptions inside `generateEvents()` may be swallowed by the fire-and-forget pattern
- The `IguazuModule.sendEvent()` wrapper has its own gating (`shouldEnableIguazuTopic`, topic allowlist) that could suppress events

**How to validate**:
- **Option A (runtime)**: Run 1% shadow prod traffic with the debug logging from PR #61377 extended to also log the *count* of events returned by each `generateEvents()` call. Compare facets-matched to events-actually-sent.
- **Option B (code audit)**: Trace each `ContainerEventsGenerator` subclass and document what conditions cause it to produce zero events. This is static analysis — no runtime needed.

### Layer 3: Event Field Completeness (NOT YET VALIDATED)

**Question**: Even when events are emitted, are all fields populated correctly?

**Known gaps**:
- Store carousels: `page_index` and `page_size` are missing (Frank flagged this explicitly)
- Possible: fields that depend on `retrievalData` lookups may fail to resolve if the store isn't found in the lookup map

**How to validate**:
- **Code audit only** — this cannot be validated at runtime without examining actual Snowflake rows. For each `ContainerEventsGenerator`, check what fields it sets on `CrossVerticalHomePageFeedEvent` and which proto fields remain at default (0, empty string, false).
- Cross-reference against the proto definition to identify fields that *should* be set but aren't.
- For fields set from `FacetV2.logging` map: verify the serializer actually populates those keys. The Jan 21 regression showed that serialization and logging are implicitly coupled — if the serializer doesn't set a key, the generator reads a default/zero.

## Proposed Approach

### Phase 1: Shadow Traffic (Layer 1 at scale + Layer 2)

Deploy Frank's debug logging branch to 1% prod traffic. Extend it to also log event counts per generator call:
```
[fz-debug] match facetId=... type=store_carousel events_generated=12
```

This validates both that facets match in the full diversity of prod traffic AND that `generateEvents()` actually produces events.

### Phase 2: Code Audit (Layer 3)

For each `ContainerEventsGenerator` subclass, document:

| Generator | Proto fields set | Proto fields NOT set | Data source for each field |
|---|---|---|---|

Known generators to audit (from PR #61377 matched types):
1. `StoreCarouselGenerator`
2. `StoreCollectionGenerator`
3. `UniversalStoreCarouselGenerator`
4. `DealCarouselGenerator`
5. `UniversalStoreFeedPlacementGenerator`
6. `FeedPlacementGenerator`
7. `FeedPlacementStandaloneUnitGenerator`
8. `UniversalItemCarouselGenerator`
9. `CarouselStandardGenerator` (multi-MX deals)
10. `UniversalTileCarouselGenerator`
11. `ItemCarouselGenerator`
12. `TileCollectionGenerator`
13. `RowStoreGenerator`
14. `RetailStoreListCarouselGenerator`
15. `ReelsCarouselGenerator`

Focus on generators that handle store-level content (1–8, 13–15). Tile generators (10, 12) are lower priority since they emit `storeId=0`.

## Priority

To be discussed at post-mortem. Frank's recommendation: leave it to core p13n team to prioritize based on impact considerations.

## References

- PR #61377: https://github.com/doordash/feed-service/pull/61377 (debug logging + coverage doc)
- PR #61283: https://github.com/doordash/feed-service/pull/61283 (fixed missing `logging` and `text.title` on FDM path wrapper FacetV2)
- PR #61366: https://github.com/doordash/feed-service/pull/61366 (sandbox Iguazu event validation)
- PR #60973: https://github.com/doordash/feed-service/pull/60973 (original store carousel fix)
