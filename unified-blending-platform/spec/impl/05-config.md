# Part 5: UnifiedExperimentConfig + UbpRuntimeUtil

## Goal

Define Kotlin data classes that model the experiment config JSON with full type safety, and
implement `UbpRuntimeUtil` which resolves the correct `ResolvedPipeline` for any
`(experimentId, treatment)` pair. This includes `extends: "control"` merge semantics (Prototype),
fallback to control for unknown treatments (Null Object), and predictor-ref resolution so steps
never receive untyped maps.

---

## Design Patterns: Factory Method + Prototype + Null Object

**Factory Method**: `UbpRuntimeUtil.resolve()` is the factory — it produces the correct
`ResolvedPipeline` for an experiment + treatment, hiding merge, cache, fallback, and
predictor-ref resolution from callers.

**Prototype**: `extends: "control"` shallow-merges a copy of the control pipeline. Treatments
that override one step param don't re-declare the rest — they prototype from control.

**Null Object**: Unknown treatment key → engine silently falls back to control. Callers never
receive null. System degrades gracefully rather than erroring.

---

## Why Typed Params (Not `Map<String, Any>`)

The previous draft used `params: Map<String, Any>` in `StepConfig`. This is brittle:
- Steps manually cast: `params["weight"] as Double` — typos are runtime errors, not compile errors
- Required fields are not enforced — missing key produces a silent null/NPE deep in a step
- No IDE completion or compiler checking on param names

**The fix:** Use `ObjectNode` for raw JSON storage (cleanly mergeable) and define a typed
`StepParams` class per step. The engine deserializes `ObjectNode → StepParams` using
`OBJECT_MAPPER_RELAX` before calling `step.process()`.

```
JSON config file
  └─ Jackson parse → TreatmentConfig (typed: PredictorConfig, PipelineConfig, ...)
       └─ UbpRuntimeUtil.resolve() → ResolvedPipeline (steps + predictors + emitTrace)
            └─ FeedRowRanker: OBJECT_MAPPER_RELAX.treeToValue(rawParams, step.paramsClass)
                 └─ step.process(rows, context, typedParams)  ← fully typed, no casts
```

Enforcements from this design:
- **Required param** (non-null Kotlin field, no default) → `MismatchedInputException` if absent from JSON
- **Optional param** (Kotlin field with default) → default used if absent, no exception
- **Unknown JSON field** → silently ignored (`FAIL_ON_UNKNOWN_PROPERTIES = false` via `OBJECT_MAPPER_RELAX`)

---

## StepParams: Sealed Interface

```kotlin
// libraries/common/.../ubp/vertical/StepParams.kt
sealed interface StepParams
```

Every step's params class implements this. The engine's `step.process()` signature takes
`StepParams`. Each step casts to its own type — guaranteed correct because the engine used
`step.paramsClass` to deserialize.

---

## Per-Step Params Classes

Params classes live in the step files (co-located with the step that owns them). Shown here for
the config spec's reference.

### ModelScoringParams

```kotlin
// In ModelScoringStep.kt
data class ModelScoringParams(
    @JsonProperty("predictor_config")
    val predictorConfig: PredictorConfig,   // required — resolved from predictor_ref by engine
) : StepParams
```

The `predictor_ref` string in config JSON is resolved by `FeedRowRanker` (not here). By the
time `step.process()` is called, `ModelScoringParams` contains the full `PredictorConfig`.

### MultiplierBoostParams

Mirrors the existing `VerticalBlendingConfig` structure exactly — because `MULTIPLIER_BOOST`
wraps `BlendingUtil.blendBundle()` which reads these same fields from
`hp_vertical_blending_config.json`. **Structural alignment is required for shadow comparison
to produce zero divergence.**

```kotlin
// In MultiplierBoostStep.kt
data class MultiplierBoostParams(
    @JsonProperty("calibration_config")
    val calibrationConfig: CalibrationConfig = CalibrationConfig(),

    @JsonProperty("intent_scoring_config")
    val intentScoringConfig: IntentScoringConfig = IntentScoringConfig(),

    @JsonProperty("vertical_boost_weights")
    val verticalBoostWeights: VerticalBoostWeights = VerticalBoostWeights(),
) : StepParams

data class CalibrationConfig(
    @JsonProperty("calibration_entries")
    val calibrationEntries: List<CalibrationEntry> = emptyList(),
    @JsonProperty("default_multiplier")
    val defaultMultiplier: Double = 1.0,
)

data class CalibrationEntry(
    @JsonProperty("vertical_ids")
    val verticalIds: List<Long> = emptyList(),
    @JsonProperty("piecewise_multipliers")
    val piecewiseMultipliers: List<PiecewiseMultiplier> = emptyList(),
)

data class PiecewiseMultiplier(
    @JsonProperty("min_score")  val minScore: Double,
    @JsonProperty("max_score")  val maxScore: Double,
    @JsonProperty("multiplier") val multiplier: Double,
)

data class IntentScoringConfig(
    @JsonProperty("feature_configs")
    val featureConfigs: List<Any> = emptyList(),       // complex nested — preserve as-is from existing config
    @JsonProperty("score_lookup_entries")
    val scoreLookupEntries: List<Any> = emptyList(),
    @JsonProperty("default_score")
    val defaultScore: Double = 1.0,
)

data class VerticalBoostWeights(
    @JsonProperty("boosting_multipliers")
    val boostingMultipliers: List<BoostingMultiplier> = emptyList(),
    @JsonProperty("default_multiplier")
    val defaultMultiplier: Double = 1.0,
    @JsonProperty("item_carousel_boosting_multipliers")
    val itemCarouselBoostingMultipliers: List<BoostingMultiplier> = emptyList(),
    @JsonProperty("item_carousel_default_multiplier")
    val itemCarouselDefaultMultiplier: Double = 1.0,
)

data class BoostingMultiplier(
    @JsonProperty("vertical_ids") val verticalIds: List<Long>,
    @JsonProperty("multiplier")   val multiplier: Double,
)
```

### DiversityRerankParams

Mirrors the existing `RerankingParams` from `hp_vertical_blending_config.json`.

```kotlin
// In DiversityRerankStep.kt
data class DiversityRerankParams(
    @JsonProperty("enabled")
    val enabled: Boolean = false,

    @JsonProperty("diversity_scoring_params")
    val diversityScoringParams: DiversityScoringParams = DiversityScoringParams(),
) : StepParams

data class DiversityScoringParams(
    @JsonProperty("weight")
    val weight: Double = 0.0,
    @JsonProperty("coefficient_for_non_rx")
    val coefficientForNonRx: Double = 1.0,
    @JsonProperty("local_window_size_for_non_rx")
    val localWindowSizeForNonRx: Int = 1,
    @JsonProperty("local_coefficient_for_non_rx")
    val localCoefficientForNonRx: Double = 1.0,
)
```

### FixedPinningParams

```kotlin
// In FixedPinningStep.kt
data class FixedPinningParams(
    @JsonProperty("rules")
    val rules: List<PinRule> = emptyList(),
) : StepParams

data class PinRule(
    @JsonProperty("row_id")
    val rowId: String? = null,           // pin specific carousel by exact ID (e.g. "pad_carousel")
    @JsonProperty("row_type")
    val rowType: String? = null,         // pin first carousel of this RowType.name
    @JsonProperty("position")
    val position: Int,                   // required — 0-indexed
    @JsonProperty("hard_pin")
    val hardPin: Boolean = true,
)
```

Validated at step entry: `require(rule.rowId != null || rule.rowType != null)`.

---

## PredictorConfig

Shared between `TreatmentConfig.predictors` and `ModelScoringParams.predictorConfig`.

```kotlin
// libraries/common/.../ubp/vertical/config/PredictorConfig.kt

@JsonIgnoreProperties(ignoreUnknown = true)
data class PredictorConfig(
    @JsonProperty("predictor_name")
    val predictorName: String,                    // required — e.g. "feed_ranking", "feed_ranking_fw"

    @JsonProperty("model_name")
    val modelName: String,                        // required — e.g. "store_ranking_v1_1"

    @JsonProperty("use_case")
    val useCase: String? = null,

    @JsonProperty("output_semantics")
    val outputSemantics: String? = null,

    @JsonProperty("features")
    val features: List<FeatureSpec> = emptyList(),

    @JsonProperty("calibration")
    val calibration: CalibrationSpec = CalibrationSpec(),
)

data class FeatureSpec(
    @JsonProperty("name")   val name: String,
    @JsonProperty("source") val source: String,
)

data class CalibrationSpec(
    @JsonProperty("type")   val type: String = "NONE",
    @JsonProperty("params") val params: ObjectNode? = null,   // flexible for PIECEWISE
)
```

---

## Data Class Definitions

### UnifiedExperimentConfig

```kotlin
// libraries/common/.../ubp/vertical/config/UnifiedExperimentConfig.kt

@JsonIgnoreProperties(ignoreUnknown = true)
data class UnifiedExperimentConfig(
    // key = treatment name ("control", "treatment_fw_v4", ...)
    val treatments: Map<String, TreatmentConfig>,
)
```

### TreatmentConfig

```kotlin
// libraries/common/.../ubp/vertical/config/TreatmentConfig.kt

@JsonIgnoreProperties(ignoreUnknown = true)
data class TreatmentConfig(
    @JsonProperty("extends")
    val extends: String? = null,

    // Named predictor configs referenced by steps via "predictor_ref"
    // Treatment overrides base predictors by name (treatment wins on key collision)
    @JsonProperty("predictors")
    val predictors: Map<String, PredictorConfig>? = null,

    // Mode 2: full pipeline override — no merge with control
    @JsonProperty("vertical_pipeline")
    val verticalPipeline: PipelineConfig? = null,

    // Mode 1: step-id → param overrides — ObjectNode for mergeable JSON tree
    @JsonProperty("step_params")
    val stepParams: Map<String, ObjectNode>? = null,

    @JsonProperty("output_config")
    val outputConfig: OutputConfig? = null,
)

data class OutputConfig(
    @JsonProperty("emit_trace")    val emitTrace: Boolean = false,
    @JsonProperty("max_feed_rows") val maxFeedRows: Int = 20,
)
```

**Why `stepParams: Map<String, ObjectNode>`** — overrides must be mergeable as a JSON tree.
`ObjectNode` lets the engine do `base.deepCopy().setAll(overrides)`. `Map<String, Any>` would
lose the nested structure needed for `vertical_boost_weights`-style nested objects.

### StepConfig

```kotlin
// libraries/common/.../ubp/vertical/config/StepConfig.kt

@JsonIgnoreProperties(ignoreUnknown = true)
data class StepConfig(
    @JsonProperty("id")
    val id: String,

    @JsonProperty("type")
    val type: String,

    // Raw JSON params — engine deserializes to typed StepParams before calling step.process()
    @JsonProperty("params")
    val rawParams: ObjectNode = ObjectNode(JsonNodeFactory.instance),
)
```

`rawParams` named to signal "pre-deserialization." JSON key stays `"params"` via `@JsonProperty`.

### PipelineConfig

```kotlin
data class PipelineConfig(
    @JsonProperty("steps")
    val steps: List<StepConfig>,
)
```

---

## Merge Semantics (Prototype Pattern)

Mode 1 (`extends`) merges at two independent levels:

### Step-level param merge

```kotlin
private fun mergeSteps(
    base: List<StepConfig>,
    stepParams: Map<String, ObjectNode>,
): List<StepConfig> = base.map { step ->
    val overrides = stepParams[step.id] ?: return@map step
    val merged = step.rawParams.deepCopy<ObjectNode>()
    merged.setAll<ObjectNode>(overrides)   // treatment top-level keys win on collision
    step.copy(rawParams = merged)
}
```

Shallow merge: if `vertical_boost_weights` is overridden, the entire object is replaced. This
is intentional — prevents partial override bugs where only some fields change but other fields
are silently inherited with stale values.

### Predictor merge

```kotlin
private fun mergePredictors(
    base: Map<String, PredictorConfig>,
    overrides: Map<String, PredictorConfig>,
): Map<String, PredictorConfig> = base + overrides   // treatment wins on key collision
```

A Mode 1 treatment can change only the predictor (e.g. new model) without touching pipeline
steps. This is the most common experiment pattern.

---

## UbpRuntimeUtil

```kotlin
// libraries/common/.../ubp/vertical/UbpRuntimeUtil.kt

class UbpRuntimeUtil(
    private val configCache: DeserializedRuntimeCache<Map<String, UnifiedExperimentConfig>>,
) {
    /**
     * Resolves the full pipeline for the given (experimentId, treatment) pair.
     *
     * Resolution order:
     *   1. Experiment not in cache          → control (Null Object)
     *   2. Treatment key not in experiment  → control (Null Object)
     *   3. verticalPipeline present         → Mode 2: return pipeline as-is
     *   4. extends present                  → Mode 1: merge steps + predictors
     *   5. Neither                          → control (defensive fallback)
     */
    fun resolve(experimentId: String, treatment: String): ResolvedPipeline {
        val config = configCache.get()?.get(experimentId)
            ?: return resolveControl(experimentId)

        val treatmentConfig = config.treatments[treatment]
            ?: return resolveControl(experimentId)

        return when {
            treatmentConfig.verticalPipeline != null -> ResolvedPipeline(
                steps      = treatmentConfig.verticalPipeline.steps,
                predictors = treatmentConfig.predictors ?: emptyMap(),
                emitTrace  = treatmentConfig.outputConfig?.emitTrace ?: false,
            )
            treatmentConfig.extends != null -> {
                val base = resolve(experimentId, treatmentConfig.extends)
                ResolvedPipeline(
                    steps      = mergeSteps(base.steps, treatmentConfig.stepParams ?: emptyMap()),
                    predictors = mergePredictors(base.predictors, treatmentConfig.predictors ?: emptyMap()),
                    emitTrace  = treatmentConfig.outputConfig?.emitTrace ?: base.emitTrace,
                )
            }
            else -> resolveControl(experimentId)
        }
    }

    private fun resolveControl(experimentId: String): ResolvedPipeline =
        resolve(experimentId, "control")

    private fun mergeSteps(
        base: List<StepConfig>,
        stepParams: Map<String, ObjectNode>,
    ): List<StepConfig> = base.map { step ->
        val overrides = stepParams[step.id] ?: return@map step
        val merged = step.rawParams.deepCopy<ObjectNode>()
        merged.setAll<ObjectNode>(overrides)
        step.copy(rawParams = merged)
    }

    /**
     * Resolves a ResolvedPipeline from an in-memory config (not from cache).
     * Used by UbpContractAssembler during shadow assembly (Part 1).
     */
    fun resolveFromConfig(config: UnifiedExperimentConfig, treatment: String): ResolvedPipeline {
        val treatmentConfig = config.treatments[treatment]
            ?: throw IllegalArgumentException("Treatment '$treatment' not in config")
        return ResolvedPipeline(
            steps      = treatmentConfig.verticalPipeline?.steps ?: emptyList(),
            predictors = treatmentConfig.predictors ?: emptyMap(),
            emitTrace  = treatmentConfig.outputConfig?.emitTrace ?: false,
        )
    }

    private fun mergePredictors(
        base: Map<String, PredictorConfig>,
        overrides: Map<String, PredictorConfig>,
    ): Map<String, PredictorConfig> = base + overrides
}

data class ResolvedPipeline(
    val steps: List<StepConfig>,
    val predictors: Map<String, PredictorConfig>,
    val emitTrace: Boolean = false,
)
```

The recursive `resolve(experimentId, "control")` is safe: the control treatment always declares
`verticalPipeline` (Mode 2) and never has `extends` — resolution terminates at Mode 2 branch.

---

## Predictor-Ref Resolution (Engine Responsibility)

The JSON config references predictors by name (`"predictor_ref": "p_act"`). Resolving this
reference — substituting the full `PredictorConfig` into the step's `rawParams` — happens in
`FeedRowRanker.rank()`, not here. This keeps `UbpRuntimeUtil` a pure config resolver.

The engine has both the step list and the predictor map from `ResolvedPipeline`, so it's the
right place for ref substitution. See Part 4 for the `resolveParams()` helper.

---

## JSON Config Examples

### Vertical control.json

Mirrors prod exactly. Seeded by reading `hp_vertical_blending_config.json` and the current
`EntityRankerConfiguration` predictor — **not hand-authored**. Shadow comparison validates
the seed is correct (target: `divergence_count = 0`).

```json
{
  "control": {
    "predictors": {
      "p_act": {
        "predictor_name": "feed_ranking",
        "model_name": "store_ranking_v1_1",
        "use_case": "USE_CASE_HOME_PAGE_VERTICAL_RANKING",
        "output_semantics": "PROBABILITY",
        "features": [
          { "name": "STORE_ID",         "source": "STORE" },
          { "name": "CONSUMER_ID",      "source": "CONSUMER" },
          { "name": "DAY_OF_WEEK",      "source": "CONTEXT" },
          { "name": "IS_DASHPASS_USER", "source": "CONSUMER" },
          { "name": "STORE_ETA",        "source": "STORE" }
        ],
        "calibration": { "type": "NONE" }
      }
    },
    "vertical_pipeline": {
      "steps": [
        { "id": "score",     "type": "MODEL_SCORING",    "params": { "predictor_ref": "p_act" } },
        {
          "id": "blend", "type": "MULTIPLIER_BOOST",
          "params": {
            "calibration_config":    { "calibration_entries": [], "default_multiplier": 1.0 },
            "intent_scoring_config": { "feature_configs": [], "score_lookup_entries": [], "default_score": 1.0 },
            "vertical_boost_weights": {
              "boosting_multipliers": [], "default_multiplier": 1.0,
              "item_carousel_boosting_multipliers": [], "item_carousel_default_multiplier": 1.0
            }
          }
        },
        {
          "id": "diversity", "type": "DIVERSITY_RERANK",
          "params": { "enabled": false, "diversity_scoring_params": { "weight": 0.0 } }
        },
        { "id": "pin", "type": "FIXED_PINNING", "params": { "rules": [] } }
      ]
    },
    "output_config": { "emit_trace": false, "max_feed_rows": 20 }
  }
}
```

### Mode 1: New model (predictor override only)

```json
{
  "treatment_fw_v4": {
    "extends": "control",
    "predictors": {
      "p_act": {
        "predictor_name": "feed_ranking_fw",
        "model_name": "store_ranker_fw_v4",
        "use_case": "USE_CASE_HOME_PAGE_VERTICAL_RANKING",
        "output_semantics": "PROBABILITY",
        "features": [
          { "name": "STORE_ID",         "source": "STORE" },
          { "name": "CONSUMER_ID",      "source": "CONSUMER" },
          { "name": "STORE_ETA",        "source": "STORE" },
          { "name": "DAY_OF_WEEK",      "source": "CONTEXT" },
          { "name": "IS_DASHPASS_USER", "source": "CONSUMER" }
        ],
        "calibration": { "type": "NONE" }
      }
    }
  }
}
```

Pipeline steps are unchanged. Only `p_act` is overridden. The `MODEL_SCORING` step's
`predictor_ref: "p_act"` resolves to this new `PredictorConfig`.

### Mode 1: Adjust NV boost weight

```json
{
  "treatment_nv_1_5x": {
    "extends": "control",
    "step_params": {
      "blend": {
        "vertical_boost_weights": {
          "boosting_multipliers": [{ "vertical_ids": [10], "multiplier": 1.5 }],
          "default_multiplier": 1.0,
          "item_carousel_boosting_multipliers": [],
          "item_carousel_default_multiplier": 1.0
        }
      }
    }
  }
}
```

The entire `vertical_boost_weights` object is replaced (shallow merge replaces the top-level
key). If only `boosting_multipliers` needs changing, the full `vertical_boost_weights` object
must be re-declared — intentional, prevents partial-override stale-value bugs.

### Mode 1: Enable diversity reranking

```json
{
  "treatment_diversity_v1": {
    "extends": "control",
    "step_params": {
      "diversity": {
        "enabled": true,
        "diversity_scoring_params": {
          "weight": 0.4,
          "coefficient_for_non_rx": 0.6,
          "local_window_size_for_non_rx": 3,
          "local_coefficient_for_non_rx": 0.3
        }
      }
    }
  }
}
```

---

## Value Function Placement in the Pipeline

The full value function `EV(c, k) = pImp(k) × pAct(c) × vAct(c)` is distributed across steps
in Phase 1 — there is no single "value function step."

| Component | Phase 1 approximation | Step |
|---|---|---|
| `pAct(c)` — P(user acts given they see carousel c) | Sibyl model output | `MODEL_SCORING` |
| `vAct(c)` — value of that action | calibration × intent × boost weight | `MULTIPLIER_BOOST` |
| `pImp(k)` — P(user sees position k) | **1.0 — not implemented** | deferred to Phase 3 |

**Step order is load-bearing:** `MODEL_SCORING` sets the base score; `MULTIPLIER_BOOST`
multiplies into it. Pipeline config should enforce this order. Phase 3 introduces an explicit
`VALUE_FUNCTION` step that computes `pImp × pAct × vAct` with position decay.

---

## Control Baseline: How UBP Compares to Prod

The shadow comparison in Part 1 is only valid if `control.json` exactly replicates the current
prod ranking output. The five requirements for `divergence_count = 0`:

| Requirement | Source of truth | How to satisfy |
|---|---|---|
| `p_act.predictor_name` matches prod | `EntityRankerConfiguration` | Read the predictor currently in use, set in `control.json` |
| `p_act.model_name` matches prod | Same | Same |
| `MULTIPLIER_BOOST` params match prod | `hp_vertical_blending_config.json` | Bootstrap from `P13nRuntimeUtil.getVerticalBlendingConfigMap()` |
| `DIVERSITY_RERANK.enabled` matches prod | `RerankingParams.enabled` in same file | Default is `false` — verify before deploying |
| `FIXED_PINNING.rules` match prod | `pinned_carousel_ranking_order.json` | Bootstrap from `PinnedCarouselUtil` |

**Bootstrap approach (from plan.md):** At init time, read the existing runtime JSON files and
construct `control.json` values from them. Do not hand-author param values — any drift between
control.json and prod config will appear as shadow divergences and must be resolved before
promoting to real traffic.

---

## Jackson Deserialization

```kotlin
// In FeedRowRanker (engine) — called before step.process():
val typedParams: StepParams = OBJECT_MAPPER_RELAX.treeToValue(resolvedRawParams, step.paramsClass)
```

`OBJECT_MAPPER_RELAX` (from `Json.kt`):
```kotlin
val OBJECT_MAPPER_RELAX: ObjectMapper = jacksonObjectMapper()
    .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
```

Hot-reload: `DeserializedRuntimeCache<Map<String, UnifiedExperimentConfig>>` with 5-minute TTL.
No pod restart needed for param changes.

---

## Files to Create

| Action | File |
|---|---|
| New | `libraries/common/.../ubp/vertical/StepParams.kt` |
| New | `libraries/common/.../ubp/vertical/config/UnifiedExperimentConfig.kt` |
| New | `libraries/common/.../ubp/vertical/config/TreatmentConfig.kt` |
| New | `libraries/common/.../ubp/vertical/config/PipelineConfig.kt` |
| New | `libraries/common/.../ubp/vertical/config/StepConfig.kt` |
| New | `libraries/common/.../ubp/vertical/config/PredictorConfig.kt` |
| New | `libraries/common/.../ubp/vertical/UbpRuntimeUtil.kt` |
| New | `ubp/experiments/vertical/control.json` (bootstrapped, not hand-authored) |

---

## Testing

```kotlin
class UbpRuntimeUtilTest {

    private val controlSteps = listOf(
        StepConfig("score",     "MODEL_SCORING",    paramNode("predictor_ref", "p_act")),
        StepConfig("blend",     "MULTIPLIER_BOOST", emptyNode()),
        StepConfig("diversity", "DIVERSITY_RERANK",  paramNode("enabled", false)),
    )
    private val controlPredictors = mapOf(
        "p_act" to PredictorConfig(predictorName = "feed_ranking", modelName = "v1")
    )
    private val baseConfig = UnifiedExperimentConfig(treatments = mapOf(
        "control" to TreatmentConfig(
            predictors = controlPredictors,
            verticalPipeline = PipelineConfig(steps = controlSteps),
        )
    ))

    @Test
    fun `resolve returns control for unknown treatment (Null Object)`() {
        val util = util(mapOf("exp1" to baseConfig))
        assertEquals(controlSteps.map { it.id }, util.resolve("exp1", "ghost").steps.map { it.id })
    }

    @Test
    fun `resolve Mode 1 merges step_params on top of control`() {
        val config = addTreatment("treatment_div", TreatmentConfig(
            extends = "control",
            stepParams = mapOf("diversity" to paramNode("enabled", true)),
        ))
        val result = util(config).resolve("exp1", "treatment_div")
        assertTrue(result.steps.first { it.id == "diversity" }.rawParams.get("enabled").booleanValue())
    }

    @Test
    fun `resolve Mode 1 merges predictors — treatment wins on collision`() {
        val config = addTreatment("treatment_fw", TreatmentConfig(
            extends = "control",
            predictors = mapOf("p_act" to PredictorConfig(predictorName = "feed_ranking_fw", modelName = "v4")),
        ))
        val result = util(config).resolve("exp1", "treatment_fw")
        assertEquals("feed_ranking_fw", result.predictors["p_act"]?.predictorName)
    }

    @Test
    fun `resolve Mode 2 returns only declared steps — does not consult control`() {
        val config = addTreatment("treatment_m2", TreatmentConfig(
            verticalPipeline = PipelineConfig(steps = listOf(StepConfig("score", "MODEL_SCORING", emptyNode()))),
        ))
        val result = util(config).resolve("exp1", "treatment_m2")
        assertEquals(1, result.steps.size)   // only the declared step, not control's 3
    }
}
```

---

## Prerequisites

- Part 4: `StepConfig` data class, `StepRegistry`
- `DeserializedRuntimeCache` — existing infrastructure
- `OBJECT_MAPPER_RELAX` from `Json.kt` — existing, used by engine for typed deserialization

---

## Done When

- All data classes parse the sample JSON correctly (Jackson deserialization unit tests pass)
- `UbpRuntimeUtil.resolve()` passes all 4 unit tests above
- `control.json` exists with all 4 steps (score → blend → diversity → pin)
- `control.json` values bootstrapped from `hp_vertical_blending_config.json` at init time
- `StepParams` sealed interface defined; all 4 `*Params` classes defined in their step files
- `MultiplierBoostParams` struct matches `VerticalBlendingConfig` field-for-field
