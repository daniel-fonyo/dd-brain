# Part 3: Ranking Step Interface and Implementations

## Goal

Define the `FeedRowRankingStep` interface (Strategy pattern) and implement the 4 concrete
ranking steps the engine dispatches to. Steps are stateless — all mutable state is on the
`FeedRow` objects. No step reads DV keys internally — all params flow through typed `StepParams`
injected by the engine after deserializing the step's `rawParams: ObjectNode`.

## Design Pattern: Strategy (Behavioral)

> "Define a family of algorithms, put each of them into a separate class, and make their objects
> interchangeable." — refactoring.guru

The engine doesn't know which algorithm it's calling — it only knows `FeedRowRankingStep`.
Swapping steps (changing the `type` key in config) requires zero engine changes.

```
FeedRowRanker
  │  dispatch
  ▼
FeedRowRankingStep.process(rows, context, params)
  │
  ├── ModelScoringStep      (type = "MODEL_SCORING")
  ├── MultiplierBoostStep   (type = "MULTIPLIER_BOOST")
  ├── DiversityRerankStep   (type = "DIVERSITY_RERANK")
  └── FixedPinningStep      (type = "FIXED_PINNING")
```

## Is-a / Has-a Analysis

```
FeedRowRankingStep  IS A strategy algorithm         → interface
ModelScoringStep    IS A FeedRowRankingStep         → implements
ModelScoringStep    HAS A EntityScorer              → DI (not inheritance)
DiversityRerankStep IS A FeedRowRankingStep         → implements (no collaborators)
FixedPinningStep    IS A FeedRowRankingStep         → implements (no collaborators)
MultiplierBoostStep IS A FeedRowRankingStep         → implements
MultiplierBoostStep HAS A BlendingUtil              → DI (wraps existing logic)
```

Steps never extend domain classes. They receive `FeedRow` (the abstraction), never
`StoreCarousel`. DI for collaborators (`EntityScorer`, `BlendingUtil`) keeps steps testable
in isolation.

## Interface Definition

```kotlin
// libraries/common/.../ubp/vertical/FeedRowRankingStep.kt
interface FeedRowRankingStep {
    val stepType: String                         // matches "type" in config steps array
    val paramsClass: Class<out StepParams>       // engine uses this to deserialize rawParams

    suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: StepParams,                      // typed — deserialized by engine before this call
    )
}
```

`params` is `StepParams` (not `Map<String, Any>`). The engine calls
`OBJECT_MAPPER_RELAX.treeToValue(rawParams, step.paramsClass)` before each `step.process()`.
Steps do a single safe cast at the top — guaranteed correct because the engine used `paramsClass`
to deserialize.

**The `params` constraint is absolute.** Steps must NOT read DV keys internally. This is the
critical abstraction boundary — experiments are config changes, not code changes.

---

## ModelScoringStep (type = "MODEL_SCORING")

```kotlin
// libraries/common/.../ubp/vertical/steps/ModelScoringStep.kt
class ModelScoringStep(
    private val entityScorer: EntityScorer,
) : FeedRowRankingStep {
    override val stepType   = "MODEL_SCORING"
    override val paramsClass = Params::class.java

    data class Params(
        @JsonProperty("predictor_config")
        val predictorConfig: PredictorConfig,   // required — resolved from predictor_ref by engine
    ) : StepParams

    override suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: StepParams,
    ) {
        val p = params as Params   // safe: engine used Params::class.java

        val scoreMap: Map<String, Double> = entityScorer.score(
            ids           = rows.map { it.id },
            predictorName = p.predictorConfig.predictorName,
            modelName     = p.predictorConfig.modelName,
            features      = p.predictorConfig.features,
            context       = context,
        )

        rows.forEach { row -> row.score = scoreMap[row.id] ?: 0.0 }
    }
}
```

`PredictorConfig` arrives fully resolved — the `predictor_ref` string from config JSON is
resolved to a `PredictorConfig` by `FeedRowRanker.resolveParams()` before this step is called.
See Part 4.

**Params contract:**
```
predictor_config             PredictorConfig   required (resolved from predictor_ref)
  .predictor_name            String            required  e.g. "feed_ranking", "feed_ranking_fw"
  .model_name                String            required  e.g. "store_ranking_v1_1"
  .features                  List<FeatureSpec> optional  default []
  .calibration               CalibrationSpec   optional  default NONE
```

---

## MultiplierBoostStep (type = "MULTIPLIER_BOOST")

Wraps `BlendingUtil.blendBundle()`. Params mirror `VerticalBlendingConfig` field-for-field —
required for shadow comparison to produce identical results to the old code path.

```kotlin
// libraries/common/.../ubp/vertical/steps/MultiplierBoostStep.kt
class MultiplierBoostStep(
    private val blendingUtil: BlendingUtil,
) : FeedRowRankingStep {
    override val stepType    = "MULTIPLIER_BOOST"
    override val paramsClass = Params::class.java

    data class Params(
        @JsonProperty("calibration_config")
        val calibrationConfig: CalibrationConfig = CalibrationConfig(),

        @JsonProperty("intent_scoring_config")
        val intentScoringConfig: IntentScoringConfig = IntentScoringConfig(),

        @JsonProperty("vertical_boost_weights")
        val verticalBoostWeights: VerticalBoostWeights = VerticalBoostWeights(),
    ) : StepParams

    override suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: StepParams,
    ) {
        val p = params as Params

        val blendingConfig = VerticalBlendingConfig(
            calibrationConfig   = p.calibrationConfig,
            intentScoringConfig = p.intentScoringConfig,
            verticalBoostWeights = p.verticalBoostWeights,
        )
        val scoreBundle = blendingUtil.blendBundle(context, rows.toScorableEntities(), blendingConfig)
        rows.forEach { row -> row.score = scoreBundle.scoreFor(row.id) }
    }
}
```

**Params contract** (all optional — defaults match "no-op / pass-through"):
```
calibration_config
  .calibration_entries          List<CalibrationEntry>   default []
  .default_multiplier           Double                   default 1.0
intent_scoring_config
  .feature_configs              List                     default []
  .score_lookup_entries         List                     default []
  .default_score                Double                   default 1.0
vertical_boost_weights
  .boosting_multipliers         List<BoostingMultiplier> default []
  .default_multiplier           Double                   default 1.0
  .item_carousel_boosting_multipliers                    default []
  .item_carousel_default_multiplier                      default 1.0
```

See Part 5 for full typed definitions of `CalibrationConfig`, `IntentScoringConfig`,
`VerticalBoostWeights`, `BoostingMultiplier`.

---

## DiversityRerankStep (type = "DIVERSITY_RERANK")

Mirrors `RerankingParams` from `hp_vertical_blending_config.json`. Starts disabled in control.

```kotlin
// libraries/common/.../ubp/vertical/steps/DiversityRerankStep.kt
class DiversityRerankStep : FeedRowRankingStep {
    override val stepType    = "DIVERSITY_RERANK"
    override val paramsClass = Params::class.java

    data class Params(
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

    override suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: StepParams,
    ) {
        val p = params as Params
        if (!p.enabled) return   // no-op — control.json default

        val d = p.diversityScoringParams
        // MMR-style: score' = (1-weight)*relevance + weight*diversity
        val selected = mutableListOf<FeedRow>()
        val typeCounts = mutableMapOf<RowType, Int>()
        val remaining = rows.toMutableList()

        rows.clear()
        while (remaining.isNotEmpty()) {
            val next = remaining.maxByOrNull { row ->
                val typeCount = typeCounts[row.type] ?: 0
                val diversity = 1.0 - (typeCount.toDouble() / (selected.size + 1).coerceAtLeast(1))
                (1 - d.weight) * row.score + d.weight * diversity * d.coefficientForNonRx
            }!!
            remaining.remove(next)
            selected.add(next)
            typeCounts[next.type] = (typeCounts[next.type] ?: 0) + 1
        }
        rows.addAll(selected)
    }
}
```

**Params contract:**
```
enabled                        Boolean   default false (no-op)
diversity_scoring_params
  .weight                      Double    default 0.0   (0 = pure relevance, 1 = pure diversity)
  .coefficient_for_non_rx      Double    default 1.0
  .local_window_size_for_non_rx Int      default 1
  .local_coefficient_for_non_rx Double   default 1.0
```

---

## FixedPinningStep (type = "FIXED_PINNING")

MLE-configured experiment pins only. Does NOT replace:
- `NonRankableHomepageOrderingUtil.updateSortOrderOfNVCarousel()` (post-checkout pin, context-driven)
- `updateSortOrderOfTasteOfDashPass()` (PAD=3, owned by its own DV)
- `updateSortOrderOfMemberPricing()` (runtime feature flag)

```kotlin
// libraries/common/.../ubp/vertical/steps/FixedPinningStep.kt
class FixedPinningStep : FeedRowRankingStep {
    override val stepType    = "FIXED_PINNING"
    override val paramsClass = Params::class.java

    data class Params(
        @JsonProperty("rules")
        val rules: List<PinRule> = emptyList(),
    ) : StepParams

    data class PinRule(
        @JsonProperty("row_id")
        val rowId: String? = null,         // pin specific carousel by exact ID
        @JsonProperty("row_type")
        val rowType: String? = null,       // pin first carousel of this RowType.name
        @JsonProperty("position")
        val position: Int,                 // required — 0-indexed
        @JsonProperty("hard_pin")
        val hardPin: Boolean = true,
    ) {
        init { require(rowId != null || rowType != null) { "PinRule requires row_id or row_type" } }
    }

    override suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: StepParams,
    ) {
        val p = params as Params
        if (p.rules.isEmpty()) return   // no-op — control.json starts with empty rules

        val pinned   = mutableMapOf<Int, FeedRow>()
        val unpinned = rows.toMutableList()

        p.rules.sortedBy { it.position }.forEach { rule ->
            val row = when {
                rule.rowId   != null -> unpinned.firstOrNull { it.id == rule.rowId }
                rule.rowType != null -> unpinned.firstOrNull { it.type.name == rule.rowType }
                else                 -> null
            } ?: return@forEach
            unpinned.remove(row)
            pinned[rule.position] = row
        }

        rows.clear()
        var idx = 0
        for (i in 0 until pinned.size + unpinned.size) {
            rows.add(pinned[i] ?: unpinned[idx++])
        }
    }
}
```

**Params contract:**
```
rules              List<PinRule>  default [] (no-op)
  .row_id          String?        optional — at least one of row_id or row_type required
  .row_type        String?        optional — RowType.name (e.g. "STORE_CAROUSEL")
  .position        Int            required — 0-indexed
  .hard_pin        Boolean        default true
```

---

## Files to Create

| File | Purpose |
|---|---|
| `libraries/common/.../ubp/vertical/FeedRowRankingStep.kt` | Interface |
| `libraries/common/.../ubp/vertical/steps/ModelScoringStep.kt` | Step + Params |
| `libraries/common/.../ubp/vertical/steps/MultiplierBoostStep.kt` | Step + Params |
| `libraries/common/.../ubp/vertical/steps/DiversityRerankStep.kt` | Step + Params |
| `libraries/common/.../ubp/vertical/steps/FixedPinningStep.kt` | Step + Params |

---

## Testing

```kotlin
@Test
fun `ModelScoringStep applies Sibyl score to rows`() {
    val scorer = mockk<EntityScorer>()
    coEvery { scorer.score(any(), "feed_ranking", "v1", any(), any()) } returns mapOf("r1" to 0.8)

    val step = ModelScoringStep(scorer)
    val rows = mutableListOf(fakeRow("r1", initialScore = 0.0))
    val params = ModelScoringStep.Params(
        predictorConfig = PredictorConfig(predictorName = "feed_ranking", modelName = "v1")
    )
    runBlocking { step.process(rows, fakeContext(), params) }
    assertEquals(0.8, rows[0].score, 0.001)
}

@Test
fun `DiversityRerankStep is no-op when disabled`() {
    val step = DiversityRerankStep()
    val rows = mutableListOf(fakeRow("s1", score = 0.9), fakeRow("s2", score = 0.5))
    runBlocking { step.process(rows, fakeContext(), DiversityRerankStep.Params(enabled = false)) }
    assertEquals(listOf("s1", "s2"), rows.map { it.id })
}

@Test
fun `DiversityRerankStep reorders when enabled with high weight`() {
    val step = DiversityRerankStep()
    val rows = mutableListOf(
        fakeRow("s1", type = RowType.STORE_CAROUSEL, score = 1.0),
        fakeRow("s2", type = RowType.STORE_CAROUSEL, score = 0.9),
        fakeRow("i1", type = RowType.ITEM_CAROUSEL,  score = 0.5),
    )
    val params = DiversityRerankStep.Params(
        enabled = true,
        diversityScoringParams = DiversityRerankStep.DiversityScoringParams(weight = 0.9),
    )
    runBlocking { step.process(rows, fakeContext(), params) }
    // With high diversity weight, ITEM_CAROUSEL should not remain last
    assertNotEquals(RowType.ITEM_CAROUSEL, rows[2].type)
}

@Test
fun `FixedPinningStep places carousel at position by row_id`() {
    val rows = mutableListOf(
        fakeRow("s1",  type = RowType.STORE_CAROUSEL),
        fakeRow("pad", type = RowType.ITEM_CAROUSEL),
        fakeRow("s2",  type = RowType.STORE_CAROUSEL),
    )
    val params = FixedPinningStep.Params(
        rules = listOf(FixedPinningStep.PinRule(rowId = "pad", position = 0))
    )
    runBlocking { FixedPinningStep().process(rows, fakeContext(), params) }
    assertEquals("pad", rows[0].id)
    assertEquals(3, rows.size)
}

@Test
fun `FixedPinningStep is no-op with empty rules`() {
    val rows = mutableListOf(fakeRow("s1"), fakeRow("s2"))
    runBlocking { FixedPinningStep().process(rows, fakeContext(), FixedPinningStep.Params()) }
    assertEquals(listOf("s1", "s2"), rows.map { it.id })
}
```

---

## Value Function Mapping

```
Full EV  = pImp(k)  ×   pAct(c)          ×   vAct(c)
Phase 1  =   1.0    ×  MODEL_SCORING     ×  MULTIPLIER_BOOST
```

- **`MODEL_SCORING`** → `pAct(c)` via Sibyl
- **`MULTIPLIER_BOOST`** → `vAct(c)` proxy via calibration × intent × boost weights (from `VerticalBlendingConfig`)
- **`pImp(k)`** = 1.0 in Phase 1 — steps have no knowledge of final position. Phase 3.
- **`DIVERSITY_RERANK`** and **`FIXED_PINNING`** are not part of the EV formula — they post-process the scored list
- **Step order is load-bearing:** `MODEL_SCORING` must precede `MULTIPLIER_BOOST`

---

## Scope Notes

**No `NV_CAROUSEL` RowType.** NV stores appear within `STORE_CAROUSEL` and `ITEM_CAROUSEL`.
NV-specific boost uses `BoostingMultiplier.verticalIds` (vertical ID matching), not a
distinct enum value.

**No `AD_CAROUSEL` RowType.** Ads are post-POC scope.

`RowType` values: `STORE_CAROUSEL, ITEM_CAROUSEL, DEAL_CAROUSEL, STORE_COLLECTION, COLLECTION_V2,
ITEM_COLLECTION, MAP_CAROUSEL, REELS_CAROUSEL, STORE_ENTITY`.

---

## Prerequisites

Part 2 (`FeedRow`, `RowType`) complete. `EntityScorer` and `BlendingUtil` are existing
interfaces — steps take them by constructor injection.

## Done When

- All 4 step classes created and unit tested
- Each step has a typed `Params` data class with `@JsonProperty` annotations — no `Map<String, Any>`
- No step reads a DV key internally
- No step casts `params["key"] as Type` — all param access through typed fields
- `MultiplierBoostStep.Params` mirrors `VerticalBlendingConfig` field-for-field
