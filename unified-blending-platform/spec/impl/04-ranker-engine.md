# Part 4: FeedRowRanker Engine + Registry

## Goal

Implement `FeedRowRanker` — the engine that resolves the step sequence for an experiment, dispatches each step, and auto-traces after each step. The engine has zero business logic — it only loops and dispatches. The step registry maps string type keys to `FeedRowRankingStep` instances.

## Design Patterns: Facade + Chain of Responsibility

**Facade (Structural)**: `FeedRowRanker.rank()` hides registry lookup, config resolution, and tracing from callers. Caller passes rows + experiment treatment and gets back a ranked list. Caller knows nothing about steps, params, or traces.

**Chain of Responsibility (Behavioral)**: Each step in the pipeline receives the full `MutableList<FeedRow>`, transforms it in-place, and passes it to the next step. Steps don't call each other — the engine drives the chain.

```
FeedRowRanker.rank(rows, experimentId, context)
  │
  ├── 1. resolve config: UbpRuntimeUtil.resolveSteps(experimentId)
  │        → List<StepConfig(type, params)>
  │
  ├── 2. for each StepConfig:
  │        step = stepRegistry[config.type] ?: skip + warn
  │        step.process(rows, context, config.params)   ← Chain of Responsibility
  │        tracer.recordStep(rows, stepId)               ← Observer (Part 7)
  │
  └── 3. return rows (now sorted by score desc)
```

## Is-a / Has-a Analysis

```
FeedRowRanker   IS A ranking facade                    → class (not interface — one implementation)
FeedRowRanker   HAS A StepRegistry                    → DI
FeedRowRanker   HAS A UbpRuntimeUtil                  → DI (config resolver)
FeedRowRanker   HAS A UbpTraceEmitter (optional)      → DI (Part 7, nullable)
StepRegistry    IS A registry of named strategies     → typealias Map<String, FeedRowRankingStep>
```

## Class Definitions

### StepConfig

```kotlin
// libraries/common/.../ubp/vertical/config/StepConfig.kt
data class StepConfig(
    val id: String,               // unique within experiment (used in traces)
    val type: String,             // maps to StepRegistry key
    val params: Map<String, Any>, // injected into step.process()
)
```

### StepRegistry

```kotlin
// libraries/common/.../ubp/vertical/StepRegistry.kt
typealias StepRegistry = Map<String, FeedRowRankingStep>

fun buildStepRegistry(
    modelScoringStep: ModelScoringStep,
    multiplierBoostStep: MultiplierBoostStep,
    diversityRerankStep: DiversityRerankStep,
    fixedPinningStep: FixedPinningStep,
): StepRegistry = mapOf(
    modelScoringStep.stepType    to modelScoringStep,
    multiplierBoostStep.stepType to multiplierBoostStep,
    diversityRerankStep.stepType to diversityRerankStep,
    fixedPinningStep.stepType    to fixedPinningStep,
)
```

Adding a new step requires implementing `FeedRowRankingStep` and adding one line here. The engine code never changes.

### UbpRuntimeUtil (stub — full spec in Part 5)

`FeedRowRanker` depends on `UbpRuntimeUtil.resolveSteps()`. Part 5 defines the full implementation. For Part 4 to compile and test, a minimal interface is sufficient:

```kotlin
// Defined fully in Part 5 — shown here for compilation contract only
interface UbpRuntimeUtil {
    fun resolveSteps(experimentId: String, treatment: String): List<StepConfig>
}
```

### FeedRowRanker

```kotlin
// libraries/common/.../ubp/vertical/FeedRowRanker.kt
class FeedRowRanker(
    private val stepRegistry: StepRegistry,
    private val runtimeUtil: UbpRuntimeUtil,
    private val traceEmitter: UbpTraceEmitter? = null,  // null until Part 7
) {
    suspend fun rank(
        rows: MutableList<FeedRow>,
        experimentId: String,
        treatment: String,
        context: RankingContext,
    ): List<FeedRow> {
        val steps = runtimeUtil.resolveSteps(experimentId, treatment)

        for (stepConfig in steps) {
            val step = stepRegistry[stepConfig.type]
            if (step == null) {
                logger.warn("Unknown step type '${stepConfig.type}' in experiment '$experimentId' — skipping")
                continue
            }
            step.process(rows, context, stepConfig.params)
            traceEmitter?.recordStep(rows, stepConfig.id, experimentId, treatment)
        }

        return rows.sortedByDescending { it.score }
    }

    companion object {
        private val logger = LoggerFactory.getLogger(FeedRowRanker::class.java)
    }
}
```

Key properties:

- **Unknown step type**: log warning, skip, continue. Never throw — a bad config key must not crash ranking.
- **traceEmitter null until Part 7**: engine compiles without tracing code. Null check is the only coupling.
- **Return sorted**: engine returns a stable sorted list by score descending. Caller does not sort.

## Files to Create

| File | Purpose |
|---|---|
| `libraries/common/.../ubp/vertical/FeedRowRanker.kt` | Engine |
| `libraries/common/.../ubp/vertical/StepRegistry.kt` | Registry typealias + builder |
| `libraries/common/.../ubp/vertical/config/StepConfig.kt` | Config data class |

## Testing

```kotlin
@Test
fun `FeedRowRanker executes steps in order`() {
    val executionOrder = mutableListOf<String>()

    val stepA = object : FeedRowRankingStep {
        override val stepType = "STEP_A"
        override suspend fun process(rows: MutableList<FeedRow>, context: RankingContext, params: Map<String, Any>) {
            executionOrder.add("A")
        }
    }
    val stepB = object : FeedRowRankingStep {
        override val stepType = "STEP_B"
        override suspend fun process(rows: MutableList<FeedRow>, context: RankingContext, params: Map<String, Any>) {
            executionOrder.add("B")
        }
    }

    val registry = mapOf("STEP_A" to stepA, "STEP_B" to stepB)
    val runtimeUtil = object : UbpRuntimeUtil {
        override fun resolveSteps(experimentId: String, treatment: String) = listOf(
            StepConfig("a", "STEP_A", emptyMap()),
            StepConfig("b", "STEP_B", emptyMap()),
        )
    }

    val ranker = FeedRowRanker(registry, runtimeUtil)
    runBlocking { ranker.rank(mutableListOf(), "exp1", "control", fakeContext()) }

    assertEquals(listOf("A", "B"), executionOrder)
}

@Test
fun `FeedRowRanker skips unknown step type with warning`() {
    val registry = emptyMap<String, FeedRowRankingStep>()
    val runtimeUtil = object : UbpRuntimeUtil {
        override fun resolveSteps(experimentId: String, treatment: String) = listOf(
            StepConfig("x", "UNKNOWN_STEP", emptyMap()),
        )
    }
    val ranker = FeedRowRanker(registry, runtimeUtil)
    // Should not throw
    assertDoesNotThrow {
        runBlocking { ranker.rank(mutableListOf(fakeRow("r1")), "exp1", "control", fakeContext()) }
    }
}

@Test
fun `FeedRowRanker returns rows sorted by score descending`() {
    val registry = emptyMap<String, FeedRowRankingStep>()
    val runtimeUtil = object : UbpRuntimeUtil {
        override fun resolveSteps(experimentId: String, treatment: String) = emptyList<StepConfig>()
    }
    val rows = mutableListOf(fakeRow("r1", score = 0.3), fakeRow("r2", score = 0.9), fakeRow("r3", score = 0.1))
    val ranker = FeedRowRanker(registry, runtimeUtil)
    val result = runBlocking { ranker.rank(rows, "exp1", "control", fakeContext()) }
    assertEquals(listOf("r2", "r1", "r3"), result.map { it.id })
}
```

## Prerequisites

- Part 2: `FeedRow`, `RowType`
- Part 3: `FeedRowRankingStep`, all 4 step implementations
- `UbpRuntimeUtil` interface (stub sufficient; full in Part 5)

## Done When

- `FeedRowRanker` compiles with a stub `UbpRuntimeUtil`
- Three unit tests pass: order, skip unknown, sort by score
- `buildStepRegistry()` wires all 4 steps
