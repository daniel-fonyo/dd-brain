# Part 4: FeedRowRanker Engine + Registry

## Goal

Implement `FeedRowRanker` — the engine that resolves the step sequence for an experiment,
deserializes typed params for each step, dispatches each step, and auto-traces after each step.
The engine has zero business logic — it only loops, resolves, and dispatches.

## Design Patterns: Facade + Chain of Responsibility

**Facade (Structural)**: `FeedRowRanker.rank()` hides registry lookup, config resolution,
predictor-ref resolution, typed params deserialization, and tracing from callers. Caller
passes rows + experiment treatment and gets back a ranked list.

**Chain of Responsibility (Behavioral)**: Each step receives the full `MutableList<FeedRow>`,
transforms it in-place, and passes it to the next step. Steps don't call each other — the
engine drives the chain.

```
FeedRowRanker.rank(rows, experimentId, treatment, context)
  │
  ├── 1. runtimeUtil.resolve(experimentId, treatment)
  │        → ResolvedPipeline(steps, predictors, emitTrace)
  │
  ├── 2. for each StepConfig:
  │        step = stepRegistry[config.type] ?: skip + warn
  │        resolvedParams = resolveParams(config.rawParams, resolved.predictors)
  │        typedParams = OBJECT_MAPPER_RELAX.treeToValue(resolvedParams, step.paramsClass)
  │        snapshot scores (if tracing)
  │        step.process(rows, context, typedParams)    ← Chain of Responsibility
  │        traceEmitter?.recordStep(...)               ← Observer (Part 7)
  │
  └── 3. return rows sorted by score descending
```

## Is-a / Has-a Analysis

```
FeedRowRanker   IS A ranking facade             → class (one implementation, no interface needed)
FeedRowRanker   HAS A StepRegistry              → DI
FeedRowRanker   HAS A UbpRuntimeUtil            → DI
FeedRowRanker   HAS A UbpTraceEmitter (opt.)    → DI (nullable, Part 7)
StepRegistry    IS A registry of strategies     → typealias Map<String, FeedRowRankingStep>
```

## Class Definitions

### StepRegistry

```kotlin
// libraries/common/.../ubp/vertical/StepRegistry.kt
typealias StepRegistry = Map<String, FeedRowRankingStep>

fun buildStepRegistry(
    modelScoringStep: ModelScoringStep,
    boostAndRankStep: BoostAndRankStep,
): StepRegistry = mapOf(
    modelScoringStep.stepType to modelScoringStep,
    boostAndRankStep.stepType to boostAndRankStep,
)
```

Adding a new step: implement `FeedRowRankingStep` + one line here. Engine never changes.

### FeedRowRanker

```kotlin
// libraries/common/.../ubp/vertical/FeedRowRanker.kt
class FeedRowRanker(
    private val stepRegistry: StepRegistry,
    private val runtimeUtil: UbpRuntimeUtil,
    private val traceEmitter: UbpTraceEmitter? = null,   // null until Part 7
) {
    /** Path 1: resolves from cache (normal path after control.json is frozen) */
    suspend fun rank(
        rows: MutableList<FeedRow>,
        experimentId: String,
        treatment: String,
        context: RankingContext,
    ): List<FeedRow> = rankInternal(rows, runtimeUtil.resolve(experimentId, treatment), context)

    /** Path 2: accepts pre-resolved pipeline (used during shadow assembly) */
    suspend fun rank(
        rows: MutableList<FeedRow>,
        resolved: ResolvedPipeline,
        context: RankingContext,
    ): List<FeedRow> = rankInternal(rows, resolved, context)

    private suspend fun rankInternal(
        rows: MutableList<FeedRow>,
        resolved: ResolvedPipeline,
        context: RankingContext,
    ): List<FeedRow> {
        for (stepConfig in resolved.steps) {
            val step = stepRegistry[stepConfig.type]
            if (step == null) {
                logger.warn("ubp_unknown_step type=${stepConfig.type} experiment=$experimentId — skipping")
                continue
            }

            // Resolve predictor_ref → full PredictorConfig inline, then deserialize to typed params
            val resolvedParams = resolveParams(stepConfig.rawParams, resolved.predictors)
            val typedParams: StepParams = OBJECT_MAPPER_RELAX.treeToValue(resolvedParams, step.paramsClass)

            val scoresBefore = if (traceEmitter != null && resolved.emitTrace) {
                rows.associate { it.id to it.score }
            } else null

            step.process(rows, context, typedParams)

            if (traceEmitter != null && scoresBefore != null) {
                traceEmitter.recordStep(rows, stepConfig.id, experimentId, treatment, context.requestId ?: "", scoresBefore)
            }
        }

        return rows.sortedByDescending { it.score }
    }

    /**
     * Resolves "predictor_ref" in rawParams by substituting the full PredictorConfig.
     * For MODEL_SCORING steps: { "predictor_ref": "p_act" } → { "predictor_config": {...} }
     * For all other steps: rawParams returned unchanged.
     */
    private fun resolveParams(
        rawParams: ObjectNode,
        predictors: Map<String, PredictorConfig>,
    ): ObjectNode {
        val ref = rawParams.get("predictor_ref")?.textValue() ?: return rawParams
        val predictor = predictors[ref]
            ?: run {
                logger.warn("ubp_unknown_predictor_ref ref=$ref — using empty PredictorConfig")
                return rawParams
            }
        val resolved = rawParams.deepCopy<ObjectNode>()
        resolved.remove("predictor_ref")
        resolved.set<ObjectNode>("predictor_config", OBJECT_MAPPER_RELAX.valueToTree(predictor))
        return resolved
    }

    companion object {
        private val logger = LoggerFactory.getLogger(FeedRowRanker::class.java)
    }
}
```

Key properties:

- **Unknown step type**: log warning, skip, continue. Never throw — a bad config key must not crash ranking.
- **Unknown predictor_ref**: log warning, return rawParams unchanged. Downstream deserialization
  will throw a `MismatchedInputException` (missing required field) which is caught by the
  shadow's `runCatching`.
- **traceEmitter null until Part 7**: engine compiles without tracing. Null check is the only
  coupling to the observer.
- **Return sorted**: engine returns stable sorted list by score descending. Callers do not sort.

---

## Files to Create

| File | Purpose |
|---|---|
| `libraries/common/.../ubp/vertical/FeedRowRanker.kt` | Engine |
| `libraries/common/.../ubp/vertical/StepRegistry.kt` | Registry typealias + builder |

`StepConfig.kt` is defined in Part 5 but referenced here — build stub first.

---

## Testing

```kotlin
@Test
fun `FeedRowRanker executes steps in order`() {
    val executionOrder = mutableListOf<String>()

    val stepA = object : FeedRowRankingStep {
        override val stepType    = "STEP_A"
        override val paramsClass = EmptyParams::class.java
        override suspend fun process(rows: MutableList<FeedRow>, context: RankingContext, params: StepParams) {
            executionOrder.add("A")
        }
    }
    val stepB = object : FeedRowRankingStep {
        override val stepType    = "STEP_B"
        override val paramsClass = EmptyParams::class.java
        override suspend fun process(rows: MutableList<FeedRow>, context: RankingContext, params: StepParams) {
            executionOrder.add("B")
        }
    }

    val registry = mapOf("STEP_A" to stepA, "STEP_B" to stepB)
    val runtimeUtil = fakeRuntimeUtil(listOf(
        StepConfig("a", "STEP_A", emptyNode()),
        StepConfig("b", "STEP_B", emptyNode()),
    ))
    val ranker = FeedRowRanker(registry, runtimeUtil)
    runBlocking { ranker.rank(mutableListOf(), "exp1", "control", fakeContext()) }

    assertEquals(listOf("A", "B"), executionOrder)
}

@Test
fun `FeedRowRanker skips unknown step type with warning — does not throw`() {
    val ranker = FeedRowRanker(emptyMap(), fakeRuntimeUtil(listOf(
        StepConfig("x", "UNKNOWN_STEP", emptyNode())
    )))
    assertDoesNotThrow {
        runBlocking { ranker.rank(mutableListOf(fakeRow("r1")), "exp1", "control", fakeContext()) }
    }
}

@Test
fun `FeedRowRanker returns rows sorted by score descending`() {
    val ranker = FeedRowRanker(emptyMap(), fakeRuntimeUtil(emptyList()))
    val rows = mutableListOf(fakeRow("r1", score = 0.3), fakeRow("r2", score = 0.9), fakeRow("r3", score = 0.1))
    val result = runBlocking { ranker.rank(rows, "exp1", "control", fakeContext()) }
    assertEquals(listOf("r2", "r1", "r3"), result.map { it.id })
}

@Test
fun `FeedRowRanker resolves predictor_ref into predictor_config`() {
    val capturedParams = mutableListOf<StepParams>()
    val mockStep = object : FeedRowRankingStep {
        override val stepType    = "MODEL_SCORING"
        override val paramsClass = ModelScoringStep.Params::class.java
        override suspend fun process(rows: MutableList<FeedRow>, context: RankingContext, params: StepParams) {
            capturedParams.add(params)
        }
    }
    val predictor = PredictorConfig(predictorName = "feed_ranking", modelName = "v1")
    val stepConfig = StepConfig("score", "MODEL_SCORING", paramNode("predictor_ref", "p_act"))
    val runtimeUtil = fakeRuntimeUtil(listOf(stepConfig), predictors = mapOf("p_act" to predictor))
    val ranker = FeedRowRanker(mapOf("MODEL_SCORING" to mockStep), runtimeUtil)

    runBlocking { ranker.rank(mutableListOf(), "exp1", "control", fakeContext()) }

    val p = capturedParams[0] as ModelScoringStep.Params
    assertEquals("feed_ranking", p.predictorConfig.predictorName)
}
```

---

## Prerequisites

- Part 2: `FeedRow`, `RowType`
- Part 3: `FeedRowRankingStep` interface + both step implementations (`ModelScoringStep`, `BoostAndRankStep`)
- Part 5: `UbpRuntimeUtil`, `ResolvedPipeline`, `StepConfig` (stub sufficient for compilation)

## Done When

- `FeedRowRanker` compiles with a stub `UbpRuntimeUtil`
- Four unit tests pass: order, skip unknown, sort by score, predictor-ref resolution
- `buildStepRegistry()` wires both steps (`ModelScoringStep`, `BoostAndRankStep`)
- `resolveParams()` substitutes `predictor_ref` → `predictor_config` before deserialization
