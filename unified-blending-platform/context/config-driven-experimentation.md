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

## Traffic Allocation & Experiment Lifecycle (Future Vision)

How does a request end up with the right config? Who manages the 100% traffic budget? How do experiments queue up and auto-promote?

### The Stack

```
┌─────────────────────────────────────────────┐
│  Experiment Manager (future)                 │
│  Owns: bucket allocation, queue, lifecycle   │
│  Writes: DV config                           │
├─────────────────────────────────────────────┤
│  DV System (exists today)                    │
│  Owns: hash-based bucketing, variant assign  │
│  Returns: treatment key in experimentMap     │
├─────────────────────────────────────────────┤
│  UbpConfigResolver (new, part of engine)     │
│  Owns: treatment key → Map<String,StepParams>│
├─────────────────────────────────────────────┤
│  Ranker (new, part of engine)                │
│  Owns: execute steps with resolved params    │
└─────────────────────────────────────────────┘
```

The DV system already handles the hard part — deterministic hash-based assignment, per-user consistency, rollout %. UBP adds the config resolution layer and the engine. The experiment manager is the future automation layer on top.

### Layer-Based Bucketing (DV Does This)

Each UBP layer gets one DV. The DV handles hash-based bucket assignment — no priority waterfall:

```
DV: ubp_vertical_ranking
  hash(consumer_id, "ubp_vertical_salt") % 1000

  Buckets 0-249:   → "treatment_fw_v4"
  Buckets 250-349: → "treatment_nv_boost"
  Buckets 350-549: → "treatment_diversity"
  Buckets 550-999: → "control"

  No priority chain — random bucket slices, peers not hierarchy.
```

A user is in **at most one** experiment per layer, but can be in experiments across layers simultaneously (different salts = independent assignment):

```
User 12345:
  ubp_vertical_ranking   → "treatment_fw_v4"   (bucket 187)
  ubp_horizontal_ranking → "control"            (bucket 723)
  Both run. Independent. Different parameters.
```

### Config Resolution (UbpConfigResolver)

The DV returns a treatment key string. The resolver turns it into typed `StepParams`:

```kotlin
class UbpConfigResolver(
    private val configCache: Map<String, Map<String, Map<String, StepParams>>>,
    //                        experimentId → treatmentKey → stepName → params
) {
    fun resolve(experimentMap: Map<String, String>): Map<String, StepParams> {
        val treatmentKey = experimentMap["ubp_vertical_ranking"] ?: "control"
        val config = configCache["ubp_vertical_ranking"]?.get(treatmentKey)
        return config ?: configCache["ubp_vertical_ranking"]?.get("control") ?: DEFAULT_CONFIG
    }
}
```

Unknown treatment key → falls back to control. System never crashes on missing config.

### Call Site (Complete Flow)

```kotlin
// In reOrderGlobalEntitiesV2:

// 1. DV system already populated experimentMap upstream
// 2. Resolve treatment key → typed step params
val config = configResolver.resolve(ctx.experimentMap)

// 3. Collect all rankable content
val items: List<Scorable> = layout.toScorables()

// 4. Rank — same pipeline, params from experiment
val ranked = ranker.rank(items, ctx, config)
```

### Traffic Management: Who Allocates the 100%?

**Today (Phase 1):** Manual. MLE tells HP eng which experiment needs what %. HP eng configures DV bucket ranges. This is fine for 1-2 experiments.

**Phase 2 — Self-service with guardrails:** MLE submits experiment config with a traffic request. An experiment manager validates (enough buckets available? layer conflict?) and writes the DV config. HP eng approves but doesn't do the work.

```yaml
# MLE submits:
experiment:
  id: exp_fw_v4
  layer: ubp_vertical_ranking
  traffic_pct: 25
  min_duration_days: 14
  config:
    treatment:
      MODEL_SCORING:
        modelName: "store_ranker_fw_v4"
      VALUE_WEIGHTING:
        govWeight: 0.7
```

Manager checks: "25% requested, 65% available → approved, assigned buckets 350-599."

**Phase 3 — Fully automated with queue and auto-ramp:**

```
Layer: ubp_vertical_ranking (1000 buckets)

RUNNING:
  exp_fw_v4:      250 buckets, started Mar 15, min 14 days
  exp_nv_boost:   100 buckets, started Mar 20, min 7 days

QUEUED (FIFO):
  1. exp_diversity:  needs 200 buckets, submitted Mar 22
  2. exp_new_model:  needs 300 buckets, submitted Mar 23

AVAILABLE: 650 buckets

── Auto-promotion ──
exp_diversity needs 200, 650 available → PROMOTE NOW
  → Assign buckets 350-549
  → Write DV config
  → Notify MLE: "Your experiment is live"

── When exp_nv_boost completes (Mar 27) ──
  Buckets 250-349 freed
  Next queued experiment auto-promotes into freed buckets
```

### Auto-Ramp (Safety)

Experiments don't go from 0% to full allocation instantly. Gradual ramp with health checks:

```
exp_diversity promoted:
  Day 1:   1% of requested (2 buckets) — monitor for crashes
  Day 2:   5% (10 buckets) — monitor core metrics
  Day 3:  25% (50 buckets) — significance starts building
  Day 5: 100% (200 buckets) — full allocation

Health check at each ramp gate:
  - p99 latency < threshold
  - error rate < threshold
  - CTR/CVR not regressing beyond guardrail
  If any fail → pause ramp, alert MLE
```

### What Needs to Be Built (and What Doesn't)

| Component | Build? | Phase |
|---|---|---|
| Hash-based bucketing | No — DV system | Exists |
| Variant assignment | No — DV system | Exists |
| Config resolution (treatment → StepParams) | Yes — `UbpConfigResolver` | Phase 1 |
| Ranking engine | Yes — `Ranker` + CoR handlers | Phase 1 |
| Bucket allocation manager | Yes — tracks allocations per layer | Phase 2 |
| Self-service experiment submission | Yes — config/YAML + validation | Phase 2 |
| Experiment queue (FIFO per layer) | Yes — auto-promote when buckets free | Phase 3 |
| Auto-ramp (gradual traffic increase) | Yes — health signal integration | Phase 3 |
| Auto-kill (metric regression → stop) | Yes — real-time metric monitoring | Phase 3 |
| Self-service UI | Yes — visual bucket allocation + queue | Phase 3 |

### Industry Context

This model follows Google's overlapping experiment infrastructure (KDD 2010):
- **Within a layer:** mutual exclusion — one experiment per user
- **Across layers:** overlap — same user in N experiments (one per layer)
- **No priority waterfall** — random bucket partitioning, experiments are peers

See `experiment-traffic-industry-research.md` for full survey (Google, Spotify, Netflix, Uber, Meta, LinkedIn, Microsoft, DoorDash current state).

---

## Relation to Other Docs

- **`poc-generic-ranking.md`** — defines `Scorable`, `RankingStep`, `RankingHandler`, `Ranker` engine. This doc shows how `StepParams` flow through that engine.
- **`mle-vertical-contract.md`** — the MLE-facing JSON config format. This doc shows the code-level mechanics that execute that config.
- **`poc-article-patterns.md`** — Composite scoring and Decorator re-ranking patterns. The `ValueWeightingStep` here is the production version of the Composite pattern from that POC.
- **`experiment-traffic-industry-research.md`** — Industry survey on traffic allocation: Google layers, Spotify bucket reuse, Netflix return-aware experimentation, DoorDash interleaving. Informs the traffic management vision above.
