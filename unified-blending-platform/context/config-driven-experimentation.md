# Config-Driven MLE Experimentation

How experiment config flows through the ranking pipeline to control model selection, value function weights, diversity enforcement, and boosting — without code changes.

## Core Idea

Every ranking operation (scoring, weighting, diversity, boosting) is a `RankingStep` that receives `StepParams` resolved from experiment config. MLE tunes params in JSON. BE ships step implementations once. The engine connects them.

```
Experiment Config (JSON)
  ↓ resolve params per step
RankingStep.execute(items, ctx, params)
  ↓ each step transforms List<Scorable>
Ranked output
```

---

## Interfaces

```kotlin
// Already in prod — all 9 carousel types implement this
interface Scorable {
    fun scorableId(): String
    val predictionScore: Double?
    fun withPredictionScore(score: Double): Scorable
}

// Marker — each step defines its own params data class
interface StepParams

// Domain logic: items in → items out. Pure function.
interface RankingStep {
    val name: String
    suspend fun execute(items: List<Scorable>, ctx: RankingContext, params: StepParams): List<Scorable>
}

// Chain of Responsibility: infrastructure (metrics, config resolution)
interface RankingHandler {
    suspend fun handle(items: List<Scorable>, ctx: RankingContext): List<Scorable>
}

abstract class BaseHandler : RankingHandler {
    var next: RankingHandler? = null
    protected suspend fun forward(items: List<Scorable>, ctx: RankingContext) =
        next?.handle(items, ctx) ?: items
}
```

---

## Step Params

Each step declares what it needs. All tunable behavior lives here — steps never read DVs directly.

```kotlin
data class ModelScoringParams(
    val modelName: String = "store_ranker_v1",
    val predictorName: String = "feed_ranking_v1",
    val useCase: String = "USE_CASE_HOME_PAGE_VERTICAL_RANKING",
) : StepParams

data class ValueWeightingParams(
    val govWeight: Double = 0.5,       // gross order value
    val fivWeight: Double = 0.3,       // first item value (new user activation)
    val strategicWeight: Double = 0.2, // strategic priority (NV, seasonal)
) : StepParams

data class DiversityParams(
    val maxConsecutiveSameType: Int = 2,   // max same-type carousels in a row
    val penaltyFactor: Double = 0.8,       // score multiplier when demoting
) : StepParams

data class BoostParams(
    val dealCarouselMultiplier: Double = 1.0,
    val boostByPositionEnabled: Boolean = false,
    val boostByPositionAllowList: List<String> = emptyList(),
) : StepParams
```

---

## Steps

Each step wraps one existing ranking operation. Same code runs for every experiment — params control behavior.

```kotlin
class ModelScoringStep(private val scorer: EntityScorer) : RankingStep {
    override val name = "MODEL_SCORING"

    override suspend fun execute(items: List<Scorable>, ctx: RankingContext, params: StepParams): List<Scorable> {
        val p = params as ModelScoringParams
        // modelName and predictorName come from experiment config
        val scores = scorer.score(ctx, items, p.predictorName, p.modelName)
        return items.map { it.withPredictionScore(scores[it.scorableId()] ?: 0.0) }
    }
}

class ValueWeightingStep : RankingStep {
    override val name = "VALUE_WEIGHTING"

    override suspend fun execute(items: List<Scorable>, ctx: RankingContext, params: StepParams): List<Scorable> {
        val p = params as ValueWeightingParams
        return items.map { item ->
            // EV(c) = pAct(c) × vAct(c)
            // Weights from experiment config control the value function mix
            val pAct = item.predictionScore ?: 0.0
            val vAct = p.govWeight * govValue(item) +
                       p.fivWeight * fivValue(item) +
                       p.strategicWeight * strategicValue(item)
            item.withPredictionScore(pAct * vAct)
        }
    }
}

class DiversityStep : RankingStep {
    override val name = "DIVERSITY"

    override suspend fun execute(items: List<Scorable>, ctx: RankingContext, params: StepParams): List<Scorable> {
        val p = params as DiversityParams
        val sorted = items.sortedByDescending { it.predictionScore }
        val result = mutableListOf<Scorable>()
        for (item in sorted) {
            val tail = result.takeLast(p.maxConsecutiveSameType)
            if (tail.size == p.maxConsecutiveSameType && tail.all { it::class == item::class }) {
                result += item.withPredictionScore((item.predictionScore ?: 0.0) * p.penaltyFactor)
            } else {
                result += item
            }
        }
        return result.sortedByDescending { it.predictionScore }
    }
}

class BoostStep(private val boostingUtil: BoostingUtil) : RankingStep {
    override val name = "BOOST"

    override suspend fun execute(items: List<Scorable>, ctx: RankingContext, params: StepParams): List<Scorable> {
        val p = params as BoostParams
        return items.map { item ->
            val multiplier = when {
                isDealCarousel(item) -> p.dealCarouselMultiplier
                else -> 1.0
            }
            item.withPredictionScore((item.predictionScore ?: 0.0) * multiplier)
        }
    }
}
```

---

## Handlers (Chain of Responsibility)

Handlers are infrastructure. They wrap steps with config resolution, metrics, and conditional execution. Steps stay pure.

```kotlin
// Resolves experiment params and passes them to the step
class ConfiguredStepHandler(
    private val step: RankingStep,
    private val config: Map<String, StepParams>,
) : BaseHandler() {
    override suspend fun handle(items: List<Scorable>, ctx: RankingContext): List<Scorable> {
        val params = config[step.name] ?: return forward(items, ctx)
        return forward(step.execute(items, ctx, params), ctx)
    }
}

// Cross-cutting: metrics without touching step logic
class MetricsHandler(
    private val name: String,
    private val metrics: MetricsClient,
    private val wrapped: RankingHandler,
) : BaseHandler() {
    override suspend fun handle(items: List<Scorable>, ctx: RankingContext): List<Scorable> {
        val start = System.nanoTime()
        val result = wrapped.handle(items, ctx)
        metrics.timer("ranking.step.$name.duration_ms", (System.nanoTime() - start) / 1_000_000)
        return forward(result, ctx)
    }
}
```

---

## Ranker

Assembles the chain. Resolves experiment config. Executes.

```kotlin
class Ranker(
    private val steps: List<RankingStep>,
    private val metrics: MetricsClient,
) {
    suspend fun rank(
        items: List<Scorable>,
        ctx: RankingContext,
        config: Map<String, StepParams>,
    ): List<Scorable> {
        val chain = steps.map { step ->
            MetricsHandler(step.name, metrics, ConfiguredStepHandler(step, config))
        }
        for (i in 0 until chain.size - 1) chain[i].next = chain[i + 1]
        return chain.firstOrNull()?.handle(items, ctx) ?: items
    }
}
```

---

## Experiment Configs

### Control (current production behavior)

```kotlin
val control = mapOf(
    "MODEL_SCORING" to ModelScoringParams(
        modelName = "store_ranker_v1",
        predictorName = "feed_ranking_v1",
    ),
    "VALUE_WEIGHTING" to ValueWeightingParams(
        govWeight = 0.5,
        fivWeight = 0.3,
        strategicWeight = 0.2,
    ),
    "DIVERSITY" to DiversityParams(
        maxConsecutiveSameType = 2,
        penaltyFactor = 0.8,
    ),
    "BOOST" to BoostParams(
        dealCarouselMultiplier = 1.0,
    ),
)
```

### Treatment A: new model + heavier GOV + stricter diversity

```kotlin
val treatmentA = mapOf(
    "MODEL_SCORING" to ModelScoringParams(
        modelName = "store_ranker_fw_v4",       // new model
        predictorName = "feed_ranking_fw_v2",   // new predictor
    ),
    "VALUE_WEIGHTING" to ValueWeightingParams(
        govWeight = 0.7,                        // heavier GOV
        fivWeight = 0.2,
        strategicWeight = 0.1,
    ),
    "DIVERSITY" to DiversityParams(
        maxConsecutiveSameType = 1,              // stricter: no consecutive same-type
        penaltyFactor = 0.6,                    // harsher penalty
    ),
    "BOOST" to BoostParams(
        dealCarouselMultiplier = 1.5,           // boost deals more
    ),
)
```

### Treatment B: same model, experiment on value function only

```kotlin
val treatmentB = mapOf(
    "MODEL_SCORING" to ModelScoringParams(
        modelName = "store_ranker_v1",          // same model as control
        predictorName = "feed_ranking_v1",
    ),
    "VALUE_WEIGHTING" to ValueWeightingParams(
        govWeight = 0.3,                        // de-emphasize GOV
        fivWeight = 0.1,
        strategicWeight = 0.6,                  // heavy strategic (NV push)
    ),
    "DIVERSITY" to DiversityParams(
        maxConsecutiveSameType = 2,              // same as control
        penaltyFactor = 0.8,
    ),
    "BOOST" to BoostParams(
        dealCarouselMultiplier = 1.0,           // same as control
    ),
)
```

---

## What It Looks Like at the Call Site

```kotlin
// 1. Resolve experiment → config
val config = when (ctx.experimentMap["ubp_vertical"]) {
    "treatment_a" -> treatmentA
    "treatment_b" -> treatmentB
    else -> control
}

// 2. Collect all rankable content into one list
val items: List<Scorable> = layout.toScorables()

// 3. Rank — same pipeline, different params
val ranked = ranker.rank(items, ctx, config)
```

Same four steps run for every user. The experiment config tunes:

| What changes | Where it lives | Example |
|---|---|---|
| Which ML model scores items | `ModelScoringParams.modelName` | `store_ranker_v1` → `store_ranker_fw_v4` |
| How value is weighted | `ValueWeightingParams.govWeight/fivWeight/strategicWeight` | GOV 0.5 → 0.7 |
| How aggressively diversity is enforced | `DiversityParams.maxConsecutiveSameType/penaltyFactor` | max 2 → max 1, penalty 0.8 → 0.6 |
| How much deals are boosted | `BoostParams.dealCarouselMultiplier` | 1.0 → 1.5 |

---

## Value Function Mapping

```
EV(c, k) = pImp(k) × pAct(c) × vAct(c)
```

| Formula component | Step | Params that tune it |
|---|---|---|
| `pAct(c)` — probability user acts | `MODEL_SCORING` | `modelName`, `predictorName` |
| `vAct(c)` — business value of action | `VALUE_WEIGHTING` | `govWeight`, `fivWeight`, `strategicWeight` |
| diversity constraint | `DIVERSITY` | `maxConsecutiveSameType`, `penaltyFactor` |
| operational boosts | `BOOST` | `dealCarouselMultiplier`, `boostByPositionEnabled` |
| `pImp(k)` — position impression probability | Future step | Not yet modeled (Phase 3) |

---

## Extension Path

### Adding a new step (BE work, one-time)

1. Define a `StepParams` data class
2. Implement `RankingStep`
3. Register in the ranker's step list

```kotlin
// Example: position decay step (future)
data class PositionDecayParams(
    val decayCurve: String = "exponential",  // "exponential", "linear", "step"
    val halfLifePosition: Int = 5,
) : StepParams

class PositionDecayStep : RankingStep {
    override val name = "POSITION_DECAY"
    override suspend fun execute(items: List<Scorable>, ctx: RankingContext, params: StepParams): List<Scorable> {
        val p = params as PositionDecayParams
        return items.mapIndexed { position, item ->
            val decay = computeDecay(position, p.decayCurve, p.halfLifePosition)
            item.withPredictionScore((item.predictionScore ?: 0.0) * decay)
        }
    }
}
```

MLE adds params to their experiment config. No code change needed after the step ships.

### Tuning existing steps (MLE work, no code)

Change JSON config values. Deploy new experiment. Done.

```json
{
  "treatment_new_weights": {
    "extends": "control",
    "step_params": {
      "VALUE_WEIGHTING": {
        "govWeight": 0.4,
        "fivWeight": 0.4,
        "strategicWeight": 0.2
      }
    }
  }
}
```

---

## Relation to Other Docs

- **`poc-generic-ranking.md`** — defines `Scorable`, `RankingStep`, `RankingHandler`, `Ranker` engine. This doc shows how `StepParams` flow through that engine.
- **`mle-vertical-contract.md`** — the MLE-facing JSON config format. This doc shows the code-level mechanics that execute that config.
- **`poc-article-patterns.md`** — Composite scoring and Decorator re-ranking patterns. The `ValueWeightingStep` here is the production version of the Composite pattern from that POC.
