# Part 6: Wiring into Pipeline

## Goal

Wire `FeedRowRanker` and `RowItemRanker` into the shadow branches added in Part 1. At this
point the shadow switches from assembly-only logging to actually running the UBP engine with
the assembled contract and comparing sort orders. No user-visible behavior changes — shadow
still returns the old result.

## What Changes

Part 1 added shadow branches with `UbpContractAssembler`:

- `DefaultHomePagePostProcessor` gated on `ubp_hp_vertical_shadow`
- `DefaultHomePageStoreRanker` gated on `ubp_hp_horizontal_shadow`

Both already assemble contracts and log them. This part adds the engine call: run
`FeedRowRanker.rank()` / `RowItemRanker.rank()` with the assembled contract, then compare
sort orders. This closes the feedback loop — you see both the contract AND the divergence.

## Architecture Decision: Dual-Path `rank()`

The engine's `rank()` method (Part 4) takes `(experimentId, treatment)` and resolves config
from the `DeserializedRuntimeCache`. But during shadow assembly, the contract is dynamically
constructed — it doesn't come from the cache.

**Solution: overloaded `rank()` on `FeedRowRanker`:**

```kotlin
// Path 1: Normal — resolves from cache (used after control.json is frozen)
suspend fun rank(
    rows: MutableList<FeedRow>,
    experimentId: String,
    treatment: String,
    context: RankingContext,
): List<FeedRow>

// Path 2: Assembly — receives pre-resolved pipeline (used during shadow assembly)
suspend fun rank(
    rows: MutableList<FeedRow>,
    resolved: ResolvedPipeline,
    context: RankingContext,
): List<FeedRow>
```

Both paths share the same internal loop (step dispatch, predictor-ref resolution, typed
params deserialization, tracing). Path 1 calls `runtimeUtil.resolve()` then delegates to the
shared internal. Path 2 skips resolution and goes straight to the shared internal.

**Add to `UbpRuntimeUtil`:**

```kotlin
/**
 * Resolves a ResolvedPipeline from an in-memory config (not from cache).
 * Used by UbpContractAssembler during shadow assembly.
 */
fun resolveFromConfig(config: UnifiedExperimentConfig, treatment: String): ResolvedPipeline {
    val treatmentConfig = config.treatments[treatment]
        ?: throw IllegalArgumentException("Treatment '$treatment' not in assembled config")
    return when {
        treatmentConfig.verticalPipeline != null -> ResolvedPipeline(
            steps      = treatmentConfig.verticalPipeline.steps,
            predictors = treatmentConfig.predictors ?: emptyMap(),
            emitTrace  = treatmentConfig.outputConfig?.emitTrace ?: false,
        )
        else -> throw IllegalArgumentException("Assembled config must have verticalPipeline")
    }
}
```

## Vertical Wiring

Part 1 already has the full shadow code. Part 6 confirms the engine call works end-to-end:

```kotlin
// In DefaultHomePagePostProcessor.reOrderGlobalEntitiesV2()
// (from Part 1 — the shadow branch, now with engine wired)
runCatching {
    // Assemble the contract from what the old path actually used
    val assembled = contractAssembler.assembleVertical(context)

    // Log it — this is the visual inspection point
    logger.info(
        "ubp_assembled_contract layer=vertical " +
        "request_id=${context.requestId} " +
        "contract=${OBJECT_MAPPER_RELAX.writeValueAsString(assembled.contract)}"
    )

    // Run UBP engine with the assembled contract
    val rows = toFeedRows(layout)
    val resolved = runtimeUtil.resolveFromConfig(assembled.contract, "control")
    val newResult = feedRowRanker.rank(rows, resolved, context)

    // Compare
    val oldOrder = oldResult.first.toSortOrderMap()
    val newOrder = newResult.toSortOrderMap()
    val divergences = oldOrder.filter { (id, pos) -> newOrder[id] != pos }

    if (divergences.isNotEmpty()) {
        logger.warn(
            "ubp_shadow_divergence layer=vertical " +
            "request_id=${context.requestId} " +
            "divergence_count=${divergences.size} " +
            "diffs=${divergences.map { (id, old) -> "$id:$old->${newOrder[id]}" }.joinToString(",")}"
        )
    } else {
        logger.info("ubp_shadow_match layer=vertical request_id=${context.requestId}")
    }
}.onFailure { logger.warn("ubp_shadow_error layer=vertical request_id=${context.requestId}", it) }
```

`toFeedRows()` and `applyRankedRows()` are the helpers introduced in Part 2.

## Horizontal Wiring

```kotlin
// In DefaultHomePageStoreRanker.rank()
// (from Part 1 — the shadow branch, now with engine wired)
if (shadowMode == "shadow") {
    runCatching {
        // Assemble per-carousel contract
        val assembled = contractAssembler.assembleHorizontal(context, collection)

        logger.info(
            "ubp_assembled_contract layer=horizontal " +
            "request_id=${context.requestId} " +
            "carousel_id=${collection.id} " +
            "ranking_type=${assembled.rankingType} " +
            "contract=${OBJECT_MAPPER_RELAX.writeValueAsString(assembled.contract)}"
        )

        val resolved = runtimeUtil.resolveFromConfig(assembled.contract, "control")
        val newResult = rowItemRanker.rank(collection, resolved, context)

        val divergences = oldResult.storeIds.zip(newResult.storeIds)
            .filter { (old, new) -> old != new }

        if (divergences.isNotEmpty()) {
            logger.warn(
                "ubp_shadow_divergence layer=horizontal " +
                "request_id=${context.requestId} " +
                "carousel_id=${collection.id} " +
                "ranking_type=${assembled.rankingType} " +
                "divergence_count=${divergences.size}"
            )
        }
    }.onFailure {
        logger.warn("ubp_shadow_error layer=horizontal request_id=${context.requestId}", it)
    }
}
```

`RowItemRanker` is the horizontal equivalent of `FeedRowRanker`. Same engine class, different
step registry. Defined in Phase 1.5.

## DI / Module Setup

Add `FeedRowRanker`, `UbpContractAssembler`, and `UbpTraceEmitter` to the DI module:

```kotlin
// In HomePageModule (or equivalent Guice/Dagger module)

@Provides @Singleton
fun provideStepRegistry(
    modelScoringStep: ModelScoringStep,
    multiplierBoostStep: MultiplierBoostStep,
    diversityRerankStep: DiversityRerankStep,
    fixedPinningStep: FixedPinningStep,
): StepRegistry = buildStepRegistry(
    modelScoringStep, multiplierBoostStep, diversityRerankStep, fixedPinningStep,
)

@Provides @Singleton
fun provideUbpRuntimeUtil(
    cache: DeserializedRuntimeCache<Map<String, UnifiedExperimentConfig>>,
): UbpRuntimeUtil = UbpRuntimeUtil(cache)

@Provides @Singleton
fun provideUbpTraceEmitter(): UbpTraceEmitter = UbpTraceEmitter()

@Provides @Singleton
fun provideFeedRowRanker(
    stepRegistry: StepRegistry,
    runtimeUtil: UbpRuntimeUtil,
    traceEmitter: UbpTraceEmitter,
): FeedRowRanker = FeedRowRanker(stepRegistry, runtimeUtil, traceEmitter)

@Provides @Singleton
fun provideContractAssembler(
    p13nRuntimeUtil: P13nRuntimeUtil,
    pinnedCarouselUtil: PinnedCarouselUtil,
    collectionScorer: CollectionScorer,
): UbpContractAssembler = UbpContractAssembler(p13nRuntimeUtil, pinnedCarouselUtil, collectionScorer)
```

`ModelScoringStep` requires `EntityScorer`, which is already provided by the existing module.

## getUbpShadowMode Helpers

Add to `DiscoveryExperimentManager`:

```kotlin
fun getUbpVerticalShadowMode(experimentMap: Map<String, String>): String =
    experimentMap[Manifest.UBP_HP_VERTICAL_SHADOW.key] ?: "off"

fun getUbpHorizontalShadowMode(experimentMap: Map<String, String>): String =
    experimentMap[Manifest.UBP_HP_HORIZONTAL_SHADOW.key] ?: "off"

fun getUbpVerticalTreatment(experimentMap: Map<String, String>): String =
    experimentMap[Manifest.UBP_HP_VERTICAL_V1.key] ?: "control"

fun getUbpHorizontalTreatment(experimentMap: Map<String, String>): String =
    experimentMap[Manifest.UBP_HP_HORIZONTAL_V1.key] ?: "control"
```

Add to `Manifest`:

```kotlin
UBP_HP_VERTICAL_SHADOW("ubp_hp_vertical_shadow"),
UBP_HP_HORIZONTAL_SHADOW("ubp_hp_horizontal_shadow"),
UBP_HP_VERTICAL_V1("ubp_hp_vertical_v1"),
UBP_HP_HORIZONTAL_V1("ubp_hp_horizontal_v1"),
```

## Transition: Assembly → Static control.json

During shadow assembly (Stage 0–1), the engine uses `rank(rows, resolved, context)` — Path 2.
Once `control.json` is frozen and validated (Stage 2+), the shadow switches to
`rank(rows, experimentId, treatment, context)` — Path 1. The assembled contract code can then
be deleted (see Part 1 Critical Rule 3: "assembler is temporary").

```
Stage 0-1 (assembly):   assembler → resolveFromConfig → rank(rows, resolved, context)
Stage 2+  (static):     configCache.get() → resolve(experimentId, treatment) → rank(rows, experimentId, treatment, context)
```

## Files to Modify

| Action | File |
|---|---|
| Modify | `FeedRowRanker.kt` — add overloaded `rank(rows, resolved, context)` |
| Modify | `UbpRuntimeUtil.kt` — add `resolveFromConfig()` |
| Modify | `DefaultHomePagePostProcessor.kt` — wire engine into shadow branch |
| Modify | `DefaultHomePageStoreRanker.kt` — wire engine into shadow branch |
| Modify | `HomePageModule.kt` (or equivalent) — add DI bindings for all UBP components |
| Modify | `DiscoveryExperimentManager.kt` — add 4 DV manifest entries + helper methods |

## Testing

Integration test: enable shadow for a test request, verify old result is returned unchanged
and divergences are logged when ranks differ.

```kotlin
@Test
fun `vertical shadow assembles contract and runs engine but returns old result`() {
    // Set shadow DV to "shadow"
    val context = fakeContext(experimentMap = mapOf(
        "ubp_hp_vertical_shadow" to "shadow",
    ))
    val response = postProcessor.reOrderGlobalEntitiesV2(context, layout)

    // Old result is returned
    assertEquals(oldRankedOrder, response.storeCarousels.map { it.id })

    // Verify assembled contract was logged
    verify { logger.info(match { it.contains("ubp_assembled_contract layer=vertical") }) }

    // Divergences logged if ranks differ — verify via log capture
}

@Test
fun `FeedRowRanker rank with ResolvedPipeline uses same dispatch as rank with experimentId`() {
    val resolved = ResolvedPipeline(
        steps = listOf(StepConfig("score", "MODEL_SCORING", paramNode("predictor_ref", "p_act"))),
        predictors = mapOf("p_act" to PredictorConfig(predictorName = "feed_ranking", modelName = "v1")),
    )
    val rows = mutableListOf(fakeRow("r1", score = 0.5))
    val result = runBlocking { ranker.rank(rows, resolved, fakeContext()) }
    assertEquals(1, result.size)
}
```

## Prerequisites

- Part 1: Shadow branch + `UbpContractAssembler` exists in both pipeline files
- Part 2: `toFeedRows()` and `applyRankedRows()` helpers defined
- Part 3: All 4 step implementations (`ModelScoringStep`, `MultiplierBoostStep`, `DiversityRerankStep`, `FixedPinningStep`)
- Part 4: `FeedRowRanker` and `buildStepRegistry()` implemented
- Part 5: `UbpRuntimeUtil`, `ResolvedPipeline`, and config data classes in place

## Done When

- `feedRowRanker.rank(rows, resolved, context)` overload exists and shares dispatch logic
- `runtimeUtil.resolveFromConfig()` converts assembled config to `ResolvedPipeline`
- Vertical shadow: assembles contract → runs engine → compares sort orders → logs divergences
- Horizontal shadow: same pattern per carousel
- DI module wires all UBP components (`FeedRowRanker`, `UbpContractAssembler`, `UbpTraceEmitter`)
- Integration test passes: old result is returned to caller
