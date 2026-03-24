# GenAI Reranker Logging — Status Report
**Date**: 2026-03-24 06:10 UTC
**Branch**: `feat/genai-reranker-logging` in feed-service
**PR**: https://github.com/doordash/feed-service/pull/62113

## Status: PR OPEN — awaiting review + CI

Sandbox-verified. PR #62113 pushed with production-ready code (7 files, +27/-2 lines). Uncommitted test/debug code remains local only.

## What's in the PR (committed)

| File | Change |
|------|--------|
| `StoreEntity.kt` | `embeddingScore: Double? = null` field + Builder |
| `ContainerEventsGenerator.kt` | `GENAI_EMBEDDING_SIMILARITY_SCORE` LoggedValue enum → `addScoreModifiers()` |
| `DomainUtilLoggingConstants.kt` | `genai_embedding_similarity_score` key constant |
| `StoreCarouselDataAdapter.kt` | `store.embeddingScore?.let { map[KEY] = ... }` |
| `StoreCarouselService.kt` | Copy embeddingScore from `generatedRecommendationStoreInfoMap` |
| `HomepageProgrammaticProductServiceUtil.kt` | Same plumbing for programmatic carousels |
| `GeneratedRecommendationCarouselService.kt` | Merge `reranker_alpha`/`reranker_beta` into `trackingPayload` |

## Sandbox Verification Evidence

Sandbox rd3987f9, pod m5xjq. Selective bootJar patching (iguazu classes only from platform, 4 packages from domain-util).

```
DEBUG_IGUAZU_SUMMARY: batch stats | {
  "topic": "cx_cross_vertical_homepage_feed",
  "total_events": 180,
  "genai_count": 110,
  "reranker_count": 110,
  "cxgen_count": 0
}
```

110/180 events (61%) contain both `genai_embedding_similarity_score` in scoreModifiers and `reranker_alpha`/`reranker_beta` in carousel_details. The 70 without are non-carousel facets.

## Next Steps

1. CI must pass (full Gradle build — verifies binary compat with StoreEntity change)
2. Code review
3. After merge: validate in Snowflake `CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE`
