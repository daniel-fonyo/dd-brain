# Part 7: Tracing

## Goal

Auto-trace after each ranking step without step code knowing about tracing. `UbpTraceEmitter` is called by the engine after every `step.process()` call when `emit_trace: true` is set in the experiment config. Steps have zero tracing code. The engine logs one structured line per (row, step) pair when `emit_trace: true`.

---

## Design Pattern: Observer (Behavioral)

> "Observer is a behavioral design pattern that lets you define a subscription mechanism to notify multiple objects about any events that happen to the object they're observing."
> — refactoring.guru

The steps are "publishers" that modify state (row scores). The engine "observes" state after each step and notifies the trace emitter. Steps have no coupling to the observer — they don't call `notifyObserver()`, they don't even know observation is happening. The engine is the intermediary.

```
Engine
  ├── step.process(rows, ...)                     ← step modifies scores (publisher, unaware)
  └── traceEmitter.recordStep(rows, stepId)       ← engine observes and notifies (observer)
       └── logger.info("ubp_trace ...")
```

---

## Is-a / Has-a Analysis

```
UbpTraceEmitter   IS A trace observer    → class
UbpTraceEmitter   HAS A Logger           → constructor-injected or companion
FeedRowRanker     HAS A UbpTraceEmitter? → optional DI (null = tracing off)
```

Steps have no reference to `UbpTraceEmitter`. This is the point — zero coupling.

---

## UbpTraceEmitter Class

```kotlin
// libraries/common/.../ubp/tracing/UbpTraceEmitter.kt
class UbpTraceEmitter(
    private val logger: Logger = LoggerFactory.getLogger(UbpTraceEmitter::class.java),
) {
    fun recordStep(
        rows: List<FeedRow>,
        stepId: String,
        experimentId: String,
        treatment: String,
        requestId: String,
        scoresBefore: Map<String, Double>,
    ) {
        rows.forEach { row ->
            logger.info(
                "ubp_trace " +
                "request_id=$requestId " +
                "experiment_id=$experimentId " +
                "treatment=$treatment " +
                "step_id=$stepId " +
                "row_id=${row.id} " +
                "row_type=${row.type.name} " +
                "score_before=${scoresBefore[row.id] ?: 0.0} " +
                "score_after=${row.score}"
            )
        }
    }
}
```

---

## How Engine Uses It

### Score Snapshot Pattern

The engine must snapshot scores BEFORE calling `step.process()`, then pass the snapshot to `recordStep()` after. Steps mutate `row.score` in-place — the snapshot captures the pre-step value.

```kotlin
// In FeedRowRanker.rank() — modified from Part 4
for (stepConfig in steps) {
    val step = stepRegistry[stepConfig.type] ?: continue.also {
        logger.warn("Unknown step type '${stepConfig.type}' — skipping")
    }

    val scoresBefore = if (traceEmitter != null && emitTrace) {
        rows.associate { it.id to it.score }
    } else null

    val resolvedParams = resolveParams(stepConfig.rawParams, resolved.predictors)
    val typedParams: StepParams = OBJECT_MAPPER_RELAX.treeToValue(resolvedParams, step.paramsClass)
    step.process(rows, context, typedParams)

    if (traceEmitter != null && scoresBefore != null) {
        traceEmitter.recordStep(rows, stepConfig.id, experimentId, treatment, context.requestId, scoresBefore)
    }
}
```

`emitTrace` is a field on `ResolvedPipeline` returned by `UbpRuntimeUtil.resolve()`.

---

## Enabling Per Experiment

Add `output_config` to `TreatmentConfig` (from Part 5):

```kotlin
data class TreatmentConfig(
    val extends: String? = null,
    val predictors: Map<String, PredictorConfig>? = null,
    val verticalPipeline: PipelineConfig? = null,
    val stepParams: Map<String, ObjectNode>? = null,
    val outputConfig: OutputConfig? = null,
)

data class OutputConfig(
    @JsonProperty("emit_trace")    val emitTrace: Boolean = false,
    @JsonProperty("max_feed_rows") val maxFeedRows: Int = 20,
)
```

In experiment JSON:

```json
{
  "treatment_fw_v4": {
    "extends": "control",
    "step_params": { },
    "output_config": { "emit_trace": true }
  }
}
```

Control has `emit_trace: false` (default). Only trace-enabled treatments pay the log volume cost.

`ResolvedPipeline.emitTrace` carries this field after `UbpRuntimeUtil.resolve()` merges configs.

---

## What This Enables

From `rfc.md` Contract 5:

- Stage-wise score snapshots: grep/query `score_before` / `score_after` per step in log tooling
- Counterfactuals: "what would rank #1 if boost weight were 1.0?"
- Opportunity cost of pinning: `score_before` on a `FIXED_PINNING` step = organic score displaced
- Validate model changes: confirm Sibyl score change actually changed `MODEL_SCORING` output
- Retire heuristics: measure their impact before removing

---

## Files to Create / Modify

| Action | File |
|---|---|
| New | `libraries/common/.../ubp/tracing/UbpTraceEmitter.kt` |
| Modify | `FeedRowRanker.kt` — add score snapshot + conditional traceEmitter call |
| Modify | `TreatmentConfig.kt` — add `outputConfig` field |
| Modify | `UbpRuntimeUtil.kt` — add `resolveEmitTrace()` |

---

## Testing

```kotlin
@Test
fun `UbpTraceEmitter logs one line per row per step`() {
    val loggedLines = mutableListOf<String>()
    val mockLogger = mockk<Logger>()
    every { mockLogger.info(capture(loggedLines)) } just Runs

    val emitter = UbpTraceEmitter(mockLogger)
    val rows = listOf(fakeRow("r1", score = 0.9), fakeRow("r2", score = 0.5))
    val scoresBefore = mapOf("r1" to 0.5, "r2" to 0.3)

    emitter.recordStep(rows, "score", "exp1", "treatment_fw", "req-123", scoresBefore)

    assertEquals(2, loggedLines.size)
    assertTrue(loggedLines.any { it.contains("row_id=r1") && it.contains("score_before=0.5") && it.contains("score_after=0.9") })
}

@Test
fun `FeedRowRanker does not call traceEmitter when null`() {
    val ranker = FeedRowRanker(registry, runtimeUtil, traceEmitter = null)
    runBlocking { ranker.rank(mutableListOf(fakeRow("r1")), "exp1", "control", fakeContext()) }
    // No exception = pass
}
```

---

## Prerequisites

- Part 4: `FeedRowRanker` (score snapshot logic modifies it)
- Part 5: `TreatmentConfig` + `UbpRuntimeUtil` (add `outputConfig`)

---

## Done When

- `UbpTraceEmitter` emits correct events per unit test
- `FeedRowRanker` snapshots scores before each step and calls emitter when non-null
- `emit_trace: true` in a treatment config activates tracing for that treatment only
