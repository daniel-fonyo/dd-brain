# Part 5: UnifiedExperimentConfig + UbpRuntimeUtil

## Goal

Define Kotlin data classes that model the experiment config JSON, and implement `UbpRuntimeUtil` which resolves the correct `List<StepConfig>` for any (experimentId, treatment) pair. This includes `extends: "control"` merge semantics (Prototype pattern) and fallback to control for unknown treatments (Null Object pattern).

---

## Design Patterns: Factory Method + Prototype + Null Object

**Factory Method (Creational)**: `UbpRuntimeUtil.resolveSteps()` is the factory method — it produces the correct `List<StepConfig>` for a given experiment + treatment, hiding all merge, cache, and fallback logic from callers.

**Prototype (Creational)**: `extends: "control"` shallow-merges a copy of the control pipeline. Treatments that override one step param don't re-declare the rest — they prototype from control.

**Null Object (Behavioral)**: When a treatment key is unknown (DV returns a value not present in the config JSON), the engine silently falls back to control. Callers never receive null. The system degrades gracefully rather than erroring.

---

## Is-a / Has-a Analysis

| Relationship | Type | Rationale |
|---|---|---|
| `UnifiedExperimentConfig` has-a `Map<String, TreatmentConfig>` | Has-a (composition) | Config is a named collection of treatment variants |
| `TreatmentConfig` has-a `PipelineConfig?` | Has-a (composition) | Treatment optionally owns a full pipeline (Mode 2) |
| `TreatmentConfig` has-a `Map<String, Map<String, Any>>?` | Has-a (composition) | Treatment optionally owns step-level param overrides (Mode 1) |
| `PipelineConfig` has-a `List<StepConfig>` | Has-a (composition) | Pipeline is an ordered sequence of steps |
| `UbpRuntimeUtil` has-a `DeserializedRuntimeCache` | Has-a (dependency injection) | Util delegates config loading to existing cache infrastructure |
| `UbpRuntimeUtil` uses `UnifiedExperimentConfig` | Uses | Reads config to resolve steps; does not own the config lifecycle |

---

## Data Class Definitions

```kotlin
// libraries/common/.../ubp/vertical/config/UnifiedExperimentConfig.kt

package com.doordash.ubp.vertical.config

import com.fasterxml.jackson.annotation.JsonProperty

/**
 * Root config for a single experiment. Keyed by experiment ID in the runtime cache.
 *
 * JSON shape:
 * {
 *   "treatments": {
 *     "control": { ... },
 *     "treatment_fw_v4": { "extends": "control", "step_params": { ... } }
 *   }
 * }
 */
data class UnifiedExperimentConfig(
    // key = treatment name (e.g. "control", "treatment_fw_v4")
    val treatments: Map<String, TreatmentConfig>,
)
```

```kotlin
// libraries/common/.../ubp/vertical/config/StepConfig.kt

package com.doordash.ubp.vertical.config

import com.fasterxml.jackson.annotation.JsonProperty

/**
 * A single ranked step in the pipeline.
 *
 * JSON shape:
 * { "id": "score", "type": "MODEL_SCORING", "params": { "model_name": "store_ranking_v1_1" } }
 */
data class StepConfig(
    @JsonProperty("id")
    val id: String,

    @JsonProperty("type")
    val type: String,

    @JsonProperty("params")
    val params: Map<String, Any> = emptyMap(),
)
```

```kotlin
// libraries/common/.../ubp/vertical/config/TreatmentConfig.kt

package com.doordash.ubp.vertical.config

import com.fasterxml.jackson.annotation.JsonProperty

/**
 * Configuration for a single treatment variant.
 *
 * Two mutually exclusive modes:
 *   Mode 1: extends + stepParams — inherits control pipeline, overrides specific step params.
 *   Mode 2: verticalPipeline    — declares complete step sequence, no merge with control.
 *
 * If both are present, verticalPipeline takes precedence (checked in UbpRuntimeUtil.resolveSteps).
 */
data class TreatmentConfig(
    // Mode 1: name of the base treatment to prototype from (typically "control")
    @JsonProperty("extends")
    val extends: String? = null,

    // Mode 2: full pipeline override — all steps declared explicitly
    @JsonProperty("vertical_pipeline")
    val verticalPipeline: PipelineConfig? = null,

    // Mode 1: step-id → param overrides (shallow-merged per step on top of base)
    @JsonProperty("step_params")
    val stepParams: Map<String, Map<String, Any>>? = null,
)
```

```kotlin
// libraries/common/.../ubp/vertical/config/PipelineConfig.kt

package com.doordash.ubp.vertical.config

import com.fasterxml.jackson.annotation.JsonProperty

/**
 * Ordered list of steps for a pipeline. Used by both control and Mode 2 treatments.
 */
data class PipelineConfig(
    @JsonProperty("steps")
    val steps: List<StepConfig>,
)
```

### Mode 1 vs Mode 2

- **Mode 1** (`extends` + `stepParams`): Inherits control pipeline. `stepParams` maps step id to override params (shallow-merged per step). Use this when only a model name or single param changes — no need to re-declare the full pipeline.
- **Mode 2** (`verticalPipeline`): Declares the complete step sequence — no merge with control. Use this when the step topology changes (e.g. adding a diversity reranking step that control does not have).

---

## Merge Semantics (Prototype)

```kotlin
private fun merge(base: List<StepConfig>, stepParams: Map<String, Map<String, Any>>): List<StepConfig> =
    base.map { step ->
        val overrides = stepParams[step.id] ?: return@map step
        step.copy(params = step.params + overrides)  // overrides win on collision
    }
```

`step.params + overrides` creates a new map — keys from overrides replace keys from base params. Keys not mentioned in overrides are preserved from base. This is shallow merge — maps within param values are NOT recursively merged. If a param value is itself a map (e.g. `row_type_multipliers`), the entire nested map must be re-declared in the override if any key within it needs changing.

---

## UbpRuntimeUtil

```kotlin
// libraries/common/.../ubp/vertical/UbpRuntimeUtil.kt

package com.doordash.ubp.vertical

import com.doordash.ubp.vertical.config.PipelineConfig
import com.doordash.ubp.vertical.config.TreatmentConfig
import com.doordash.ubp.vertical.config.UnifiedExperimentConfig
import com.doordash.runtime.DeserializedRuntimeCache

/**
 * Resolves the correct List<StepConfig> for a given (experimentId, treatment) pair.
 *
 * Applies Factory Method, Prototype, and Null Object patterns:
 *   - Factory Method: resolveSteps() produces the correct list, hiding merge/cache/fallback logic.
 *   - Prototype:      Mode 1 treatments inherit control's pipeline via shallow merge.
 *   - Null Object:    Unknown treatment keys fall back to control — callers never receive null.
 */
class UbpRuntimeUtil(
    // Cache holds Map<experimentId, UnifiedExperimentConfig>. TTL: 5 minutes. Hot-reloaded.
    private val configCache: DeserializedRuntimeCache<Map<String, UnifiedExperimentConfig>>,
) {
    /**
     * Resolves the step list for the given experiment and treatment.
     *
     * Resolution order:
     *   1. Experiment not found in cache              → control steps (Null Object)
     *   2. Treatment key not found in experiment      → control steps (Null Object)
     *   3. verticalPipeline present (Mode 2)          → return pipeline steps directly
     *   4. extends present (Mode 1)                   → resolve base, shallow-merge stepParams
     *   5. Neither field set                          → control steps (defensive fallback)
     */
    fun resolveSteps(experimentId: String, treatment: String): List<StepConfig> {
        val config = configCache.get()[experimentId]
            ?: return controlSteps(experimentId)

        val treatmentConfig = config.treatments[treatment]
            ?: return controlSteps(experimentId)  // Null Object: unknown treatment → control

        return when {
            treatmentConfig.verticalPipeline != null -> {
                // Mode 2: full pipeline override — return steps as-is
                treatmentConfig.verticalPipeline.steps
            }
            treatmentConfig.extends != null -> {
                // Mode 1: prototype from base treatment, apply step param overrides
                val base = resolveSteps(experimentId, treatmentConfig.extends)
                val overrides = treatmentConfig.stepParams ?: emptyMap()
                merge(base, overrides)
            }
            else -> controlSteps(experimentId)
        }
    }

    /**
     * Returns control steps for the given experiment.
     * Recursive call to resolveSteps with "control" is safe — "control" never has `extends`.
     */
    private fun controlSteps(experimentId: String): List<StepConfig> =
        resolveSteps(experimentId, "control")

    /**
     * Shallow-merges stepParams overrides on top of the base pipeline.
     * Steps not mentioned in stepParams are returned unchanged.
     * Within a step, only top-level param keys are merged — nested maps are not recursed.
     */
    private fun merge(base: List<StepConfig>, stepParams: Map<String, Map<String, Any>>): List<StepConfig> =
        base.map { step ->
            val overrides = stepParams[step.id] ?: return@map step
            step.copy(params = step.params + overrides)
        }
}
```

The recursive call to `resolveSteps(experimentId, "control")` is safe because the control treatment is always defined with `verticalPipeline` and never has an `extends` field — resolution terminates at the Mode 2 branch.

---

## JSON Examples

### control.json (abbreviated)

```json
{
  "control": {
    "vertical_pipeline": {
      "steps": [
        {
          "id": "score",
          "type": "MODEL_SCORING",
          "params": {
            "predictor_name": "FEED_RANKING_SIBYL_PREDICTOR_NAME",
            "model_name": "store_ranking_v1_1",
            "calibration_multiplier": 1.0
          }
        },
        {
          "id": "blend",
          "type": "MULTIPLIER_BOOST",
          "params": {
            "row_type_multipliers": {
              "STORE_CAROUSEL": 1.0,
              "NV_CAROUSEL": 0.9
            }
          }
        },
        {
          "id": "pin",
          "type": "FIXED_PINNING",
          "params": {
            "pins": []
          }
        }
      ]
    }
  }
}
```

### Mode 1 treatment (param override)

```json
{
  "treatment_fw_v4": {
    "extends": "control",
    "step_params": {
      "score": { "model_name": "store_ranker_fw_v4" }
    }
  }
}
```

Result: the full control pipeline is used, but the `score` step's `model_name` is replaced with `store_ranker_fw_v4`. The `predictor_name` and `calibration_multiplier` keys are preserved from control because they are not mentioned in `step_params`.

### Mode 2 treatment (full pipeline)

```json
{
  "treatment_diversity_v1": {
    "vertical_pipeline": {
      "steps": [
        {
          "id": "score",
          "type": "MODEL_SCORING",
          "params": {
            "predictor_name": "FEED_RANKING_SIBYL_PREDICTOR_NAME",
            "model_name": "store_ranking_v1_1",
            "calibration_multiplier": 1.0
          }
        },
        {
          "id": "blend",
          "type": "MULTIPLIER_BOOST",
          "params": {
            "row_type_multipliers": {
              "STORE_CAROUSEL": 1.0,
              "NV_CAROUSEL": 0.9
            }
          }
        },
        {
          "id": "diversity",
          "type": "DIVERSITY_RERANK",
          "params": {
            "weight": 0.4
          }
        },
        {
          "id": "pin",
          "type": "FIXED_PINNING",
          "params": {
            "pins": []
          }
        }
      ]
    }
  }
}
```

Result: a 4-step pipeline. The `diversity` step is inserted between `blend` and `pin`. Control is not consulted — the entire step list is taken from `vertical_pipeline`.

---

## JSON → Kotlin Deserialization

Config files are hot-reloaded via `DeserializedRuntimeCache` (existing infrastructure). The cache holds `Map<String, UnifiedExperimentConfig>` keyed by experiment ID. TTL: 5 minutes. No pod restart needed for param changes during a running experiment.

Deserialization uses Jackson directly — no manual map lookups or brittle key parsing:

```kotlin
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.fasterxml.jackson.module.kotlin.readValue

private val mapper = jacksonObjectMapper()

// Called by cache loader — raw JSON string → typed config
fun parseConfig(json: String): UnifiedExperimentConfig =
    mapper.readValue<UnifiedExperimentConfig>(json)
```

Snake_case JSON fields map to camelCase Kotlin fields via `@JsonProperty` annotations on each field (as shown in the data classes above). Alternatively, a Jackson `PropertyNamingStrategies.SNAKE_CASE` naming strategy configured on the `ObjectMapper` eliminates the need for per-field annotations. Either approach yields the same typed result — no manual `map["key"]` lookups.

---

## Files to Create

| Action | File |
|---|---|
| New | `libraries/common/.../ubp/vertical/config/UnifiedExperimentConfig.kt` |
| New | `libraries/common/.../ubp/vertical/config/TreatmentConfig.kt` |
| New | `libraries/common/.../ubp/vertical/config/PipelineConfig.kt` |
| New | `libraries/common/.../ubp/vertical/UbpRuntimeUtil.kt` |
| New | `ubp/experiments/control.json` |

---

## Testing

```kotlin
import com.doordash.ubp.vertical.UbpRuntimeUtil
import com.doordash.ubp.vertical.StepConfig
import com.doordash.ubp.vertical.config.PipelineConfig
import com.doordash.ubp.vertical.config.TreatmentConfig
import com.doordash.ubp.vertical.config.UnifiedExperimentConfig
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

class UbpRuntimeUtilTest {

    /** Minimal in-memory stand-in for DeserializedRuntimeCache. */
    private class FakeCache(private val data: Map<String, UnifiedExperimentConfig>) {
        fun get(): Map<String, UnifiedExperimentConfig> = data
    }

    @Test
    fun `resolveSteps returns control steps for unknown treatment (Null Object)`() {
        val config = mapOf(
            "exp1" to UnifiedExperimentConfig(
                treatments = mapOf(
                    "control" to TreatmentConfig(
                        verticalPipeline = PipelineConfig(
                            steps = listOf(StepConfig("s1", "MODEL_SCORING", emptyMap()))
                        )
                    )
                )
            )
        )
        val cache = FakeCache(config)
        val util = UbpRuntimeUtil(cache)
        val steps = util.resolveSteps("exp1", "nonexistent_treatment")
        assertEquals(listOf("s1"), steps.map { it.id })
    }

    @Test
    fun `resolveSteps merges step_params override on top of control (Mode 1)`() {
        val baseParams = mapOf("model_name" to "v1", "predictor_name" to "sib")
        val config = mapOf(
            "exp1" to UnifiedExperimentConfig(
                treatments = mapOf(
                    "control" to TreatmentConfig(
                        verticalPipeline = PipelineConfig(
                            steps = listOf(StepConfig("score", "MODEL_SCORING", baseParams))
                        )
                    ),
                    "treatment_fw" to TreatmentConfig(
                        extends = "control",
                        stepParams = mapOf("score" to mapOf("model_name" to "v2"))
                    ),
                )
            )
        )
        val cache = FakeCache(config)
        val util = UbpRuntimeUtil(cache)
        val steps = util.resolveSteps("exp1", "treatment_fw")
        assertEquals("v2", steps[0].params["model_name"])       // override applied
        assertEquals("sib", steps[0].params["predictor_name"])  // preserved from control
    }

    @Test
    fun `resolveSteps returns full pipeline for Mode 2 treatment`() {
        val config = mapOf(
            "exp1" to UnifiedExperimentConfig(
                treatments = mapOf(
                    "control" to TreatmentConfig(
                        verticalPipeline = PipelineConfig(
                            steps = listOf(StepConfig("score", "MODEL_SCORING", emptyMap()))
                        )
                    ),
                    "treatment_diversity" to TreatmentConfig(
                        verticalPipeline = PipelineConfig(
                            steps = listOf(
                                StepConfig("score", "MODEL_SCORING", emptyMap()),
                                StepConfig("diversity", "DIVERSITY_RERANK", mapOf("weight" to 0.4)),
                            )
                        )
                    ),
                )
            )
        )
        val cache = FakeCache(config)
        val util = UbpRuntimeUtil(cache)
        val steps = util.resolveSteps("exp1", "treatment_diversity")
        assertEquals(2, steps.size)
        assertEquals("DIVERSITY_RERANK", steps[1].type)
    }
}
```

---

## Prerequisites

- Part 4: `StepConfig` data class — referenced in `PipelineConfig.steps` and in merge logic
- `DeserializedRuntimeCache` — existing infrastructure; provides hot-reloaded config with 5-minute TTL

---

## Done When

- `UnifiedExperimentConfig` and sub-classes parse the sample JSON correctly (verified via a deserialization unit test or manual inspection with Jackson)
- `UbpRuntimeUtil.resolveSteps()` passes all 3 unit tests above
- `control.json` is created with the 3-step control pipeline (score → blend → pin) matching the shapes in rfc.md
