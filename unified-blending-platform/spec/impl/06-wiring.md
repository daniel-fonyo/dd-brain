# Part 6: Wiring into Pipeline

## Goal

Wire `FeedRowRanker` into `DefaultHomePagePostProcessor` (vertical) and `DefaultHomePageStoreRanker` (horizontal) behind the shadow DVs added in Part 1. At this point the shadow DVs switch from stub no-ops to the real engine. No user-visible behavior changes — shadow still returns the old result.

## What Changes

Part 1 added shadow branches to both files:

- `DefaultHomePagePostProcessor` gated on `ubp_hp_vertical_shadow`
- `DefaultHomePageStoreRanker` gated on `ubp_hp_horizontal_shadow`

Both contained `// TODO: wire real ranker` stubs. This part replaces those stubs with real `FeedRowRanker` and `RowItemRanker` calls.

## Vertical Wiring

Replace the stub in `DefaultHomePagePostProcessor`:

```kotlin
// Before (Part 1 stub):
runCatching {
    val rows = toFeedRows(layout)
    val newResult = feedRowRanker.rank(rows, "control", context)  // stub
    ubpShadowEmitter.emit(...)
}

// After (Part 6 — same call site, real ranker now injected):
runCatching {
    val rows = toFeedRows(layout)
    val treatment = discoveryExperimentManager.getUbpVerticalTreatment(context.experimentMap)
    val newResult = feedRowRanker.rank(rows, "ubp_hp_vertical", treatment, context)
    val ranked = applyRankedRows(newResult, layout)
    ubpShadowEmitter.emit(
        requestId = context.requestId,
        layer = "vertical",
        oldSortOrders = oldResult.first.toSortOrderMap(),
        newSortOrders = ranked.toSortOrderMap(),
    )
}
```

`toFeedRows()` and `applyRankedRows()` are the helpers introduced in Part 2. `feedRowRanker` is constructor-injected (see DI / Module Setup).

## Horizontal Wiring

Replace the stub in `DefaultHomePageStoreRanker`:

```kotlin
// Before (Part 1 stub):
if (shadowMode == "shadow") {
    runCatching {
        val newResult = rowItemRanker.rank(collection, "control", context)  // stub
        ubpShadowEmitter.emit(...)
    }
}

// After (Part 6 — stub replaced with real RowItemRanker; horizontal engine mirrors vertical):
if (shadowMode == "shadow") {
    runCatching {
        val treatment = discoveryExptManager.getUbpHorizontalTreatment(context.experimentMap)
        val newResult = rowItemRanker.rank(collection, "ubp_hp_horizontal", treatment, context)
        ubpShadowEmitter.emit(
            requestId = context.requestId,
            layer = "horizontal",
            carouselId = collection.id,
            oldStoreOrder = oldResult.storeIds,
            newStoreOrder = newResult.storeIds,
        )
    }.onFailure { logger.warn("UBP shadow horizontal failed", it) }
}
```

`RowItemRanker` is the horizontal equivalent of `FeedRowRanker`. Same engine class, different step registry. Defined in horizontal Phase 2 — at this point it mirrors the vertical structure as a stub.

## DI / Module Setup

Add `FeedRowRanker` to the DI module that wires `DefaultHomePagePostProcessor`:

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
fun provideFeedRowRanker(
    stepRegistry: StepRegistry,
    runtimeUtil: UbpRuntimeUtil,
): FeedRowRanker = FeedRowRanker(stepRegistry, runtimeUtil)
```

`ModelScoringStep` requires `EntityScorer`, which is already provided by the existing module.

## getUbpVerticalTreatment Helper

Add to `DiscoveryExperimentManager`:

```kotlin
fun getUbpVerticalTreatment(experimentMap: Map<String, String>): String =
    experimentMap[Manifest.UBP_HP_VERTICAL_V1.key] ?: "control"

fun getUbpHorizontalTreatment(experimentMap: Map<String, String>): String =
    experimentMap[Manifest.UBP_HP_HORIZONTAL_V1.key] ?: "control"
```

Add to `Manifest`:

```kotlin
UBP_HP_VERTICAL_V1("ubp_hp_vertical_v1"),
UBP_HP_HORIZONTAL_V1("ubp_hp_horizontal_v1"),
```

## Files to Modify

| Action | File |
|---|---|
| Modify | `DefaultHomePagePostProcessor.kt` — replace stub with real `feedRowRanker.rank()` |
| Modify | `DefaultHomePageStoreRanker.kt` — replace stub with real `rowItemRanker.rank()` |
| Modify | `HomePageModule.kt` (or equivalent) — add DI bindings for `StepRegistry`, `UbpRuntimeUtil`, `FeedRowRanker` |
| Modify | `DiscoveryExperimentManager.kt` — add `getUbpVerticalTreatment()`, `getUbpHorizontalTreatment()`, and two manifest entries |

## Testing

Integration test: enable shadow for a test request, verify shadow emitter is called and old result is returned unchanged.

```kotlin
@Test
fun `vertical shadow runs new ranker but returns old result`() {
    // Set shadow DV to "shadow"
    val context = fakeContext(experimentMap = mapOf(
        "ubp_hp_vertical_shadow" to "shadow",
        "ubp_hp_vertical_v1" to "control",
    ))
    val response = postProcessor.reOrderGlobalEntitiesV2(context, layout)

    // Old result is returned
    assertEquals(oldRankedOrder, response.storeCarousels.map { it.id })

    // Emitter was called (if ranks differ)
    verify { ubpShadowEmitter.emit(any(), any(), any(), any()) }
}
```

## Prerequisites

- Part 1: Shadow branch exists in both `DefaultHomePagePostProcessor` and `DefaultHomePageStoreRanker`
- Part 2: `toFeedRows()` and `applyRankedRows()` helpers defined
- Part 3: All 4 step implementations (`ModelScoringStep`, `MultiplierBoostStep`, `DiversityRerankStep`, `FixedPinningStep`)
- Part 4: `FeedRowRanker` and `buildStepRegistry()` implemented
- Part 5: `UbpRuntimeUtil` and `control.json` config in place

## Done When

- `feedRowRanker.rank()` is called in the vertical shadow branch (no longer a stub)
- `rowItemRanker.rank()` is called in the horizontal shadow branch (mirrors vertical shape)
- Shadow emitter receives real comparison data from both branches
- Integration test passes: old result is returned to caller, emitter is invoked with ranked output
