# GenAI Reranker Logging — Analysis Report

**Date**: 2026-03-24 05:55 UTC
**Branch**: `feat/genai-reranker-logging` in feed-service
**Sandbox**: rd3987f9, pod `m5xjq`

## Result: SUCCESS

The `cx_cross_vertical_homepage_feed` Iguazu events now contain:
- **`genai_embedding_similarity_score`** as a score modifier (110/180 events = 61%)
- **`reranker_alpha` and `reranker_beta`** in `carousel_details` JSON (110/180 events = 61%)

### Key Log Evidence

```
DEBUG_IGUAZU_ENTER: topic=cx_cross_vertical_homepage_feed
DEBUG_IGUAZU_GENERATE: topic=cx_cross_vertical_homepage_feed
DEBUG_IGUAZU_COUNT: topic=cx_cross_vertical_homepage_feed count=180
DEBUG_IGUAZU_SUMMARY: batch stats | {
  "topic": "cx_cross_vertical_homepage_feed",
  "total_events": 180,
  "genai_count": 110,
  "reranker_count": 110,
  "cxgen_count": 0
}
DEBUG_IGUAZU_SENT: sent 180 events for topic=cx_cross_vertical_homepage_feed
```

- **180 total events** per homepage load (all carousels x stores)
- **110 events (61%)** have `genai_embedding` in their `scoreModifiersList`
- **110 events (61%)** have `reranker` data in their `carouselDetails`
- **0 cxgen events** — expected, test consumer has no GenAI carousels from Content Systems
- 70 events (39%) are store-list or non-carousel facets that don't have carousel-level tracking

### What Was Deployed

Two modules patched into the base image bootJar using selective class injection:

**Platform module (3 iguazu classes):**
- `IguazuModule.kt` — Debug logging for cx_cross_vertical_homepage_feed events, sandbox bypass (`isSandboxEnv = false`)

**Domain-util module (337 classes from 4 packages):**
- `ContainerEventsGenerator.kt` — Added `GENAI_EMBEDDING_SIMILARITY_SCORE` LoggedValue enum entry
- `DomainUtilLoggingConstants.kt` — Added `genai_embedding_similarity_score` constant key
- `StoreCarouselDataAdapter.kt` — Hardcoded `genai_embedding_similarity_score = 0.42` in logging map (test value)
- `StoreCarouselDataAdapterUtil.kt` — Injected `reranker_alpha: 0.5, reranker_beta: 1.5` when trackingPayload empty

### Build Strategy

Full build fails due to unrelated modules (`:notification`, `:offers`, `:verticalpage`). Solution: **selective class-level bootJar patching**.

1. Build `:platform` and `:domain-util` modules only (both compile fine)
2. Extract original JARs from base image bootJar
3. Inject ONLY changed package classes into the original JARs
4. Patch bootJar with hybrid JARs (our changes + base image's StoreEntity for binary compat)

Key constraint: `StoreEntity.embeddingScore` field CANNOT be deployed via patching because it changes the Builder constructor signature, causing `NoSuchMethodError` in other modules. Instead, the score is injected directly in `StoreCarouselDataAdapter` logging map.

### All Iguazu Topics Observed (doortest tenant)

| Topic | Count | Description |
|-------|-------|-------------|
| ads_sl_tracking_event | 10 | Ads tracking |
| home_page_store_ranker_compact_event | 7 | Store ranking |
| cx_home_page_feed_carousel_event | 3 | Carousel events |
| feed_service_grpc_performance | 2 | gRPC perf metrics |
| cx_home_page_feed_carousel_capping_event | 1 | Capping |
| vertical_entry_point_event | 1 | Vertical tabs |
| smart_suggestion_event_load | 1 | Search suggestions |
| home_page_pagination_event | 1 | Pagination |
| **cx_cross_vertical_homepage_feed** | **1** | **Our target** |
| reorder_carousel_create_event | 1 | Reorder carousel |

### What "genai_count: 110" and "reranker_count: 110" Mean

- **genai_count**: Number of events where `scoreModifiersList` contains an entry with name containing "genai_embedding". Value = 0.42 (hardcoded test value).
- **reranker_count**: Number of events where `carouselDetails` JSON string contains "reranker". Values = `reranker_alpha: 0.5, reranker_beta: 1.5` (hardcoded test values).
- The 70 events without these fields are store-list facets or facets where `StoreCarouselDataAdapterUtil` wasn't called.

### What's Still Needed for Production

1. **Remove test hardcodes** — Replace `0.42` with real `store.embeddingScore` from Content Systems
2. **Remove reranker injection** — Real reranker values should come from `trackingPayload`
3. **Add `embeddingScore` to StoreEntity** — Requires full build (no patching). PR must pass CI.
4. **Remove debug logging** — Strip `DEBUG_IGUAZU_*` logs from IguazuModule
5. **Re-enable sandbox Iguazu gating** — Change `isSandboxEnv = false` back to `PlatformEvars.isSandboxEnv()`
6. **Validate in Snowflake** — After merging, verify events appear in `CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE` with `genai_embedding_similarity_score` in score_modifiers and reranker data in carousel_details
