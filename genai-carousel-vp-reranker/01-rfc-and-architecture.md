# RFC & Architecture: GenAI Carousel Horizontal Reranker

## RFC Summary

**Authors**: Dipali, Ranjan, Yu Zhang | **Date**: 2026-03-20

### What is EBR?

**Embedding-Based Retrieval**: carousel title + metadata are turned into a text embedding; merchant/item profiles are also embedded; k-NN cosine similarity retrieves the most relevant stores/items per personalized carousel. Computed **online (real-time)**, not offline.

Conceptually: embedding score = **relevance signal**, store ranker score = **conversion signal (pConv)**.

### Current Reranker (Block Reranking)

`reRankStoresByScoreAndSimilarity` in `GeneratedRecommendationCarouselService`:

1. **Read coefficients** from DV per experiment variant: `alpha`, `beta`, `k`
2. **Global sort** by EBR cosine similarity (descending)
3. **Block rerank**: split into chunks of size `k`, within each chunk compute:
   ```
   combinedScore = finalScore^alpha * embeddingScore^beta
   ```
   Sort each chunk by combinedScore descending, stitch back in order.
4. **Return** reordered `LiteStoreCollection`

**Production defaults**: `alpha=0.0`, `beta=1.0`, `k=10` (validated via A/B).
With `alpha=0.0`, `finalScore^0 = 1` — ranker score drops out. Ranking is 100% EBR similarity today.

**Why blocks, not global sort?** Blocks preserve coarse EBR ordering — a low-similarity store can't leapfrog into a higher block.

### Proposed Change (VP Value Function)

Replace `finalScore` with an explicit VP score from raw components in `ScoreWithFactors`:

```kotlin
// Current
val combinedScore = finalScore.pow(alpha) * similarity.pow(beta)

// Proposed
val pConv       = scoreWithFactors?.rawScorerScores ?: 0.0
val pDasherCost = scoreWithFactors?.expectedDasherCost ?: 1.0
val pCommission = scoreWithFactors?.expectedCommission ?: 1.0
val vpScore     = pConv * pDasherCost * pCommission
val combinedScore = vpScore.pow(alpha) * similarity.pow(beta)
```

### Online A/B Experiment

| Arm | Signal | Description |
|-----|--------|-------------|
| Control | pConv only (`rawScorerScores`) | Clean baseline, no VP multipliers |
| T1 | `pConv * pDasherCost * pCommission` (v1 coefficients) | Full VP formula, set 1 |
| T2 | `pConv * pDasherCost * pCommission` (v2 coefficients) | Full VP formula, set 2 |

---

## Architecture

### Two Retrieval Paths

There are two code paths that fetch GenAI carousels. They diverge at **fetch time** and converge at **collection building**. **Both paths produce Iguazu events** — event emission happens at response serialization time in `HomepageResponseSerializer`, long after the paths have merged.

| Path | What It Does | Original trackingPayload? | embeddingScore? |
|------|-------------|--------------------------|----------------|
| A: Direct fetcher | Runs EBR retrieval in-process inside feed-service | Yes (`carousel_rank`, `day_part`, `last_update_date`) | Yes |
| B: Content Systems RPC | Calls `ContentSystemsFeedService.GetGeneratedRecommendationCarousels()` gRPC to a separate service (`web-content-systems`) that does the EBR retrieval | **No** — `trackingPayload` not in `content_systems.proto` (defaults to `emptyMap()`) | **Yes** — `GeneratedRecommendationStoreInfo.embedding_score` (field 3) IS in the proto |

**Why Path B exists**: Architectural separation — EBR retrieval logic (calling embedding models, k-NN, store decoration) runs in a dedicated Content Systems service. Feed-service handles ranking, logging, and response serialization. Path B is the default for most traffic.

**Key insight for our work**: All three new logging fields work on **both paths** (100% of traffic):
- **`genai_embedding_similarity_score`**: `embeddingScore` is in the content systems proto → deserialized by `GeneratedRecommendationContentSystemsFetcherUtil` → flows through `storeInfoMap` → `StoreEntity.genaiEmbeddingSimilarityScore` → Iguazu event.
- **`genai_reranker_alpha` / `genai_reranker_beta`**: Injected post-convergence in `rerankStoreByBlockReranking()` via `collection.copy(trackingPayload += {alpha, beta})`. Alpha/beta come from DV coefficients, not the fetch path.

**What IS missing on Path B (pre-existing gap)**: The *original* `trackingPayload` fields from the fetch phase (`carousel_rank`, `day_part`, `last_update_date`) — these are set in `createGeneratedRecommendationCarousels()` but not serialized in `content_systems.proto`. For Path B, `carousel_details` JSON will contain `{carousel_id, genai_reranker_alpha, genai_reranker_beta}` but NOT `carousel_rank` or `day_part`.

Switch controlled by: `DiscoveryExperimentManager.shouldUseGeneratedRecommendationEBROption2(experimentMap)`

---

## Full Call Stacks

### Shared Entry Point

```
HomepagePipeline.constructWorkflow()
  └─ programmaticProductsJob
       └─ HomepageProgrammaticProductRegistry.registerAllProducts()
            └─ registers HomepageGeneratedRecommendationProductName
                 if shouldEnableGeneratedRecommendationProduct(experimentMap)
       └─ DiscoveryFlow.runFlow()
            └─ BaseDynamicProgrammaticProductService.process()
                 └─ preRankJob  → preRank()  → preRankImpl()
                 │    └─ fetch()          ← PATHS DIVERGE HERE
                 │    └─ contextDecorate()
                 │    └─ contentDecorate()
                 │    └─ getProductList()
                 │    └─ dynamicPreFilter()  → buildCollection()
                 └─ rankJob     → rank()     → DiscoveryRanker.rank()
                 └─ postFilterJob → dynamicPostFilter()
```

### Path A: Direct Fetcher

```
HomepageGeneratedRecommendationProductService.fetch()
  └─ fetcher.fetch(context)
       └─ GeneratedRecommendationFetcher.fetch()
            ├─ storeListFetchJob.fetch()                    ← fetch store entities from search
            ├─ GeneratedRecommendationCarouselService
            │    .getGeneratedRecommendationDataListFromDB()
            │    └─ if EBR treatment:
            │    │    GeneratedRecommendationWorkflowService
            │    │      .fetchCarouselsWithEbrStores()
            │    │      ├─ getRetrievalContext()
            │    │      ├─ getCarouselsForCxEBR()           ← CRDB EBR table
            │    │      ├─ extractCarouselEmbeddings()
            │    │      ├─ getEBRStoreIdsAndScoresWithLocation()  ← EBR model call → embeddingScore
            │    │      ├─ getEBRItemIdsMap()                     ← EBR item model
            │    │      └─ createGeneratedRecommendationCarousels()
            │    │           → sets storeInfoMap[storeId].embeddingScore
            │    │           → sets trackingPayload = {carousel_rank, day_part, last_update_date}
            │    └─ else:
            │         GeneratedRecommendationWorkflowService
            │           .fetchCarouselsWithPreComputedStores()
            │           ├─ getCarouselsForCx()              ← CRDB pre-computed table
            │           └─ createGeneratedRecommendationCarousels()
            │                → embeddingScore from CRDB storeIdsAndEmbeddingScoreMap
            │                → trackingPayload set same as above
            └─ filterGeneratedRecommendationDataList()
```

### Path B: Content Systems RPC

```
HomepageGeneratedRecommendationProductService.fetch()
  └─ contentSystemsFetcher.fetch(context)
       └─ GeneratedRecommendationContentSystemsFetcher.fetch()
            └─ ContentSystemsRepository.fetchGeneratedRecommendationCarousels()
                 └─ ContentSystemsClient.getGeneratedRecommendationCarousels()
                      └─ gRPC to web-content-systems service
                           └─ [SERVER SIDE]
                              ContentSystemsController.getGeneratedRecommendationCarousels()
                              └─ GeneratedRecommendationCarouselsPipeline.constructWorkflow()
                                   ├─ requestToContextTransformer.transform()
                                   ├─ retriever.getQueryEmbeddings()       ← carousel embeddings from CRDB
                                   ├─ retriever.retrieve()                 ← EBR store/item models
                                   ├─ contentSystemsStoreDecorationService.decorate()
                                   ├─ createGeneratedRecommendationCarousels()
                                   │    → embeddingScore + trackingPayload set (same as Path A)
                                   └─ GeneratedRecommendationCarouselsSerializer.serialize()
                                        → embeddingScore: SERIALIZED (in proto)
                                        → trackingPayload: NOT SERIALIZED (not in proto)
            └─ GeneratedRecommendationContentSystemsFetcherUtil
                 .toGeneratedRecommendationData(response)
                 → embeddingScore: DESERIALIZED from proto
                 → trackingPayload: defaults to emptyMap()
```

### Shared Path (After Fetch Converges)

```
HomepageGeneratedRecommendationProductService.getProductList()
  └─ GeneratedRecommendationCarouselService.generateHomepageProduct()
       └─ creates HomepageGeneratedRecommendationProduct per carousel

HomepageGeneratedRecommendationProduct.buildCollection()
  └─ LiteStoreCollection.Builder
       .generatedRecommendationStoreInfoMap(data.storeInfoMap)  ← embeddingScore here
       .trackingPayload(data.trackingPayload)                    ← carousel_rank, day_part
       .build()

DiscoveryRanker.rank()
  └─ populates collection.storePredictionScoresMap              ← ScoreWithFactors per store

HomepageProgrammaticCarouselGenerationService.generate()
  └─ postFilterGeneratedRecommendationCollections()
       └─ HomepageGeneratedRecommendationProduct.getPostFilterResult()
            ├─ image contextualization from generatedRecommendationStoreInfoMap
            ├─ store filtering (empty headers, delivery unavailable, etc.)
            ├─ GeneratedRecommendationCarouselService                    ★ RERANKER
            │    .reRankStoresByScoreAndSimilarity()
            │    └─ rerankStoreByBlockReranking()
            │         ├─ reads alpha, beta, k from DV
            │         ├─ builds storeSimilarityEmbeddingMap from data.storeInfoMap
            │         ├─ combinedScore = finalScore^alpha * embeddingScore^beta
            │         └─ returns collection.copy(entities=reranked, trackingPayload+=alpha,beta)  ★ OUR CHANGE
            └─ deduplication (Option 2 only)
  └─ updateStoreEntity()                                         ★ COPIES embeddingScore TO StoreEntity
       └─ copies ScoreWithFactors fields + embeddingScore to StoreEntity
  └─ generateStoreCarousel()
       └─ StoreCarousel.Builder
            .trackingPayload(collection.trackingPayload)         ← flows alpha/beta through
            .build()
```

### Logging & Iguazu Event Emission

```
StoreCarouselDataAdapter.toFacetV2()
  └─ StoreCarouselDataAdapterUtil.generateLogging()              ← CONTAINER level
       └─ if trackingPayload.isNotEmpty():
            carousel_details = JSON({carousel_id, ...trackingPayload})   ★ alpha/beta land here
  └─ StoreCarouselDataAdapter.generateStoreLogging()             ← CHILD/STORE level
       └─ map[EXPECTED_COMMISSION_KEY] = store.expectedCommission
       └─ map[EXPECTED_DASHER_COST_KEY] = store.expectedDasherCost
       └─ map[EMBEDDING_SIMILARITY_SCORE_KEY] = store.embeddingScore    ★ OUR CHANGE

SerializerHelperUtil.sendHomePageFeedEventLogging()
  └─ IguazuEventUtil.generateXVerticalHomePageFeedSegmentEvents()
       └─ processFilteredSections()
            └─ ContainerEventsGenerator.StoreCarouselGenerator.generateEvents()
                 └─ per store:
                      additionalContainerLogging()
                        └─ LoggedValue.CAROUSEL_DETAILS reads carousel_details from struct
                      additionalChildLogging()
                        └─ LoggedValue.EMBEDDING_SIMILARITY_SCORE                ★ OUR CHANGE
                             reads from struct, calls builder.addScoreModifiers()
                      addEvent(builder.build())
  └─ IguazuModule.sendEvent()
       └─ proxyEventSender.send(event)                           → Kafka → Snowflake
```

### Key Files Reference

| File (feed-service) | Key Methods |
|------|---------|
| `libraries/discovery/.../GeneratedRecommendationCarouselService.kt` | `rerankStoreByBlockReranking()` (line ~138), `reRankStoresByScoreAndSimilarity()` (line ~117) |
| `libraries/discovery/.../HomepageGeneratedRecommendationProduct.kt` | `buildCollection()` (line ~136), `getPostFilterResult()` (line ~53) |
| `libraries/discovery/.../HomepageGeneratedRecommendationProductService.kt` | `fetch()` (line ~47), `rank()` (line ~132) |
| `libraries/discovery/.../fetcher/GeneratedRecommendationFetcher.kt` | `fetch()` (line ~44) — Path A |
| `libraries/discovery/.../fetcher/GeneratedRecommendationContentSystemsFetcher.kt` | `fetch()` (line ~30) — Path B |
| `libraries/domain-util/.../contentsystems/GeneratedRecommendationDataService.kt` | `createGeneratedRecommendationCarousels()` (line ~524) |
| `libraries/domain-util/.../contentsystems/GeneratedRecommendationWorkflowService.kt` | `fetchCarouselsWithEbrStores()` (line ~145), `fetchCarouselsWithPreComputedStores()` (line ~27) |
| `libraries/domain-util/.../carousel/models/collections/LiteStoreCollection.kt` | Data class — `generatedRecommendationStoreInfoMap` (line 92), `trackingPayload` (line 94) |
| `pipelines/homepage/.../HomepageProgrammaticCarouselGenerationService.kt` | `generate()` (line ~64), `generateStoreCarousel()` (line ~561) |
| `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt` | `LoggedValue` enum (line ~1848), `StoreCarouselGenerator` (line ~396) |
| `libraries/domain-util/.../facets/adapters/StoreCarouselDataAdapter.kt` | `generateStoreLogging()` (line ~2337) |
| `libraries/domain-util/.../carousel/utils/StoreCarouselDataAdapterUtil.kt` | `generateLogging()` (line ~494) — writes `carousel_details` |

### What's Already Logged vs The Gap

| Proto Field | # | What It Logs | Status |
|-------------|---|--------------|--------|
| `horizontal_element_score` | 19 | VP prediction score per store | Logged |
| `raw_horizontal_element_score` | 80 | Raw Sibyl score pre-multipliers | Logged |
| `facet_score` | 20 | Carousel-level aggregate score | Logged |
| `score_modifiers` | 59 | Named score components | Logged (adding `genai_embedding_similarity_score` — **both paths**) |
| `carousel_details` | 81 | JSON carousel metadata | Logged — `genai_reranker_alpha`/`beta` added post-convergence (**both paths**). Path A also has `carousel_rank`, `day_part`; Path B does not (pre-existing proto gap). |
| `predictor_names` / `model_ids` | 57/58 | Model names and IDs | Logged |
| **EBR embedding similarity score** | — | Per-store cosine similarity | **NOT LOGGED → score_modifiers** |
| **Alpha, Beta** | — | Reranker formula exponents | **NOT LOGGED → carousel_details** |
