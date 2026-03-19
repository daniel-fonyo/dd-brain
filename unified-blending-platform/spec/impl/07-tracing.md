# Part 7: Tracing

## Goal

Auto-trace after each ranking step without step code knowing about tracing. `UbpTraceEmitter` is called by the engine after every `step.process()` call when `emit_trace: true` is set in the experiment config. Steps have zero tracing code. The engine emits one Iguazu event per (row, step) pair.

---

## Design Pattern: Observer (Behavioral)

> "Observer is a behavioral design pattern that lets you define a subscription mechanism to notify multiple objects about any events that happen to the object they're observing."
> — refactoring.guru

The steps are "publishers" that modify state (row scores). The engine "observes" state after each step and notifies the trace emitter. Steps have no coupling to the observer — they don't call `notifyObserver()`, they don't even know observation is happening. The engine is the intermediary.

```
Engine
  ├── step.process(rows, ...)                     ← step modifies scores (publisher, unaware)
  └── traceEmitter.recordStep(rows, stepId)       ← engine observes and notifies (observer)
       └── iguazuModule.emit(UbpFeedRowRankingTrace(...))
```

---

## Is-a / Has-a Analysis

```
UbpTraceEmitter   IS A trace observer    → class
UbpTraceEmitter   HAS A IguazuModule     → DI
FeedRowRanker     HAS A UbpTraceEmitter? → optional DI (null = tracing off)
```

Steps have no reference to `UbpTraceEmitter`. This is the point — zero coupling.

---

## Proto Definition

```protobuf
// services-protobuf: ubp_feed_row_ranking_trace.proto
syntax = "proto3";

message UbpFeedRowRankingTrace {
  string request_id     = 1;
  string experiment_id  = 2;
  string treatment      = 3;
  string step_id        = 4;   // matches StepConfig.id
  string row_id         = 5;
  string row_type       = 6;   // RowType.name
  double score_before   = 7;
  double score_after    = 8;
  int64  timestamp_ms   = 9;
}
```

One event per (row, step) pair. For a 20-row feed with 3 steps: 60 events per request. Emit only when `emit_trace: true` — never for control traffic.

---

## UbpTraceEmitter Class

```kotlin
// libraries/common/.../ubp/tracing/UbpTraceEmitter.kt
class UbpTraceEmitter(
    private val iguazuModule: IguazuModule,
) {
    fun recordStep(
        rows: List<FeedRow>,
        stepId: String,
        experimentId: String,
        treatment: String,
        requestId: String,
        scoresBefore: Map<String, Double>,  // snapshot taken before step ran
    ) {
        val now = System.currentTimeMillis()
        rows.forEach { row ->
            iguazuModule.emit(UbpFeedRowRankingTrace(
                requestId = requestId,
                experimentId = experimentId,
                treatment = treatment,
                stepId = stepId,
                rowId = row.id,
                rowType = row.type.name,
                scoreBefore = scoresBefore[row.id] ?: 0.0,
                scoreAfter = row.score,
                timestampMs = now,
            ))
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

    step.process(rows, context, stepConfig.params)

    if (traceEmitter != null && scoresBefore != null) {
        traceEmitter.recordStep(rows, stepConfig.id, experimentId, treatment, context.requestId, scoresBefore)
    }
}
```

`emitTrace` is resolved from the experiment config output config via `UbpRuntimeUtil.resolveEmitTrace()`.

---

## Enabling Per Experiment

Add `output_config` to `TreatmentConfig` (from Part 5):

```kotlin
data class TreatmentConfig(
    val extends: String? = null,
    val verticalPipeline: PipelineConfig? = null,
    val stepParams: Map<String, Map<String, Any>>? = null,
    val outputConfig: OutputConfig? = null,  // NEW
)

data class OutputConfig(
    val emitTrace: Boolean = false,
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

Control has `emit_trace: false` (default). Only trace-enabled treatments pay the Iguazu volume cost.

`UbpRuntimeUtil.resolveEmitTrace(experimentId, treatment): Boolean` reads this field after merging.

---

## What This Enables

From `rfc.md` Contract 5:

- Stage-wise score snapshots: query `score_before` / `score_after` per step in Snowflake
- Counterfactuals: "what would rank #1 if boost weight were 1.0?"
- Opportunity cost of pinning: `score_before` on a `FIXED_PINNING` step = organic score displaced
- Validate model changes: confirm Sibyl score change actually changed `MODEL_SCORING` output
- Retire heuristics: measure their impact before removing

---

## Files to Create / Modify

| Action | File |
|---|---|
| New | `libraries/common/.../ubp/tracing/UbpTraceEmitter.kt` |
| New | `services-protobuf/ubp_feed_row_ranking_trace.proto` |
| Modify | `FeedRowRanker.kt` — add score snapshot + conditional traceEmitter call |
| Modify | `TreatmentConfig.kt` — add `outputConfig` field |
| Modify | `UbpRuntimeUtil.kt` — add `resolveEmitTrace()` |

---

## Testing

```kotlin
@Test
fun `UbpTraceEmitter emits one event per row per step`() {
    val emittedEvents = mutableListOf<UbpFeedRowRankingTrace>()
    val iguazu = mockk<IguazuModule>()
    every { iguazu.emit(any()) } answers { emittedEvents.add(firstArg()) }

    val emitter = UbpTraceEmitter(iguazu)
    val rows = listOf(fakeRow("r1", score = 0.9), fakeRow("r2", score = 0.5))
    val scoresBefore = mapOf("r1" to 0.5, "r2" to 0.3)

    emitter.recordStep(rows, "score", "exp1", "treatment_fw", "req-123", scoresBefore)

    assertEquals(2, emittedEvents.size)
    val r1Event = emittedEvents.first { it.rowId == "r1" }
    assertEquals(0.5, r1Event.scoreBefore, 0.001)
    assertEquals(0.9, r1Event.scoreAfter, 0.001)
    assertEquals("score", r1Event.stepId)
}

@Test
fun `FeedRowRanker does not call traceEmitter when null`() {
    val ranker = FeedRowRanker(registry, runtimeUtil, traceEmitter = null)
    // Should not throw, emitter never called
    runBlocking { ranker.rank(mutableListOf(fakeRow("r1")), "exp1", "control", fakeContext()) }
}
```

---

## Prerequisites

- Part 4: `FeedRowRanker` (score snapshot logic modifies it)
- Part 5: `TreatmentConfig` + `UbpRuntimeUtil` (add `outputConfig`)
- `IguazuModule` — existing infrastructure

---

## Done When

- Proto file created and compiled
- `UbpTraceEmitter` emits correct events per unit test
- `FeedRowRanker` snapshots scores before each step and calls emitter when non-null
- `emit_trace: true` in a treatment config activates tracing for that treatment only
