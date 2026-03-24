# GenAI Reranker Logging — Status Report
**Date**: 2026-03-24 05:55 UTC (2026-03-23 22:55 PST)
**Branch**: `feat/genai-reranker-logging` in feed-service

## Summary
**VERIFIED WORKING.** The `genai_embedding_similarity_score` and `reranker_alpha/beta` fields are flowing through the Iguazu `cx_cross_vertical_homepage_feed` events. 110 of 180 events (61%) contain both fields per homepage load.

## What's Deployed (sandbox rd3987f9)

### Selective bootJar patching (Makefile `build-remote-devbox-server`)
Full build fails on unrelated modules. We compile `:platform` and `:domain-util` only, then inject specific class packages into the base image bootJar.

**Platform (3 classes — iguazu package only):**
1. `IguazuModule.kt` — Debug logging + sandbox bypass (`isSandboxEnv = false`)

**Domain-util (337 classes — 4 packages):**
2. `ContainerEventsGenerator.kt` — `GENAI_EMBEDDING_SIMILARITY_SCORE` LoggedValue enum
3. `DomainUtilLoggingConstants.kt` — `genai_embedding_similarity_score` key constant
4. `StoreCarouselDataAdapter.kt` — Hardcoded `genai_embedding_similarity_score = 0.42` in logging map
5. `StoreCarouselDataAdapterUtil.kt` — Reranker alpha/beta injection when trackingPayload empty

**NOT deployed (binary compat constraint):**
- `StoreEntity.embeddingScore` field — changes Builder constructor signature, breaks other modules
- `HomepageRequestToContext.kt` consumer ID override — in homepage pipeline, not patched
- `HomepageProgrammaticProductServiceUtil.kt` embeddingScore — same reason as StoreEntity

## Key Evidence

```
DEBUG_IGUAZU_SUMMARY: batch stats | {
  "topic": "cx_cross_vertical_homepage_feed",
  "total_events": 180,
  "genai_count": 110,     # events with genai_embedding score modifier
  "reranker_count": 110,  # events with reranker in carousel_details
  "cxgen_count": 0         # no GenAI carousels for test consumer (expected)
}
```

## Build Strategy

```
1. gradle :platform:jar          → compiles IguazuModule + StoreEntity (with embeddingScore)
2. gradle :domain-util:jar       → compiles ContainerEventsGenerator, DataAdapter, etc.
3. Extract ORIGINAL platform.jar from bootJar
4. Inject ONLY iguazu/*.class into original platform.jar (preserves base image StoreEntity)
5. Patch bootJar with hybrid platform.jar
6. Extract ORIGINAL domain-util.jar from bootJar
7. Inject changed packages (iguazu, carousel/utils, facets/adapters, utils/logging)
8. Patch bootJar with hybrid domain-util.jar
```

This avoids the `NoSuchMethodError: StoreEntity$Builder.<init>` that occurs when the full platform.jar is patched.

## Next Steps for PR

1. Remove test hardcodes (`0.42`, reranker `0.5/1.5`)
2. Remove debug logging (`DEBUG_IGUAZU_*`)
3. Re-enable sandbox Iguazu gating
4. The PR should add `embeddingScore` to StoreEntity + plumb it from Content Systems response
5. CI full build will verify binary compat
