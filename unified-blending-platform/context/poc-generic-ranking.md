# Ranking Abstraction POC

## The Problem

The homepage ranking pipeline has 9 vertical placement types and 3+ horizontal entity types. They all have `id` and `predictionScore`, but no shared interface. Ranking logic is scattered across `BaseEntityRankerConfiguration`, `StoreCarouselService`, and a dozen utility classes, each operating on different concrete types. There's no uniform way to score, reorder, or experiment across them.

## The Scorable Interface

We introduce one interface: `Scorable`. It captures the minimal contract that every rankable item already satisfies.

```kotlin
interface Scorable {
    val id: String
    val predictionScore: Double?
    fun withPredictionScore(score: Double): Scorable
}
```

Three members. That's it. Every domain type that participates in ranking already has `id` and `predictionScore` as fields — implementing `Scorable` is adding `override` to existing fields and a one-line `withPredictionScore` via `copy()`.

### Why inheritance over composition?

We considered three approaches:

| Approach | What it means | Verdict |
|---|---|---|
| **Inheritance (is-a)** | Domain objects implement `Scorable` directly | **Chosen** — fields already exist, one-line change per class, polymorphic `withPredictionScore` eliminates `when` blocks |
| **Composition (has-a)** | Wrap items in `Ranked<T>(item, id, score)` | Rejected — reconstructs what inheritance gives us for free, reintroduces `when` blocks for wrapping/unwrapping |
| **Registry (strategy)** | Register `ScoreExtractor<T>` per type | Rejected — 12 extractor objects doing `it.id` / `it.predictionScore` / `it.copy(...)`, more boilerplate than `override` |

**Inheritance wins because the fields already exist.** We're not inventing a hierarchy — we're formalizing a convention (`id` + `predictionScore`) that every rankable type already follows into a compile-time contract.

### Scorable vs SortablePlacement

`SortablePlacement` already exists in feed-service. It handles **vertical positioning** (sort order, gap rules, immersive spacing). `Scorable` handles **scoring** (ML prediction scores, blending, boosting). They're orthogonal capabilities:

```
Scorable                          SortablePlacement
"I can be scored"                 "I can be positioned vertically"
  id, predictionScore,              sortOrder, assignSortOrder(),
  withPredictionScore()             getRequiredGap(), getMaxVerticalPlacement()

        ┌─── Vertical types implement BOTH ───┐
        │  StoreCarousel    : Scorable, SortablePlacement
        │  ItemCarousel     : Scorable, SortablePlacement
        │  DealCarousel     : Scorable, SortablePlacement
        │  CollectionV2     : Scorable, SortablePlacement
        │  ItemCollection   : Scorable, SortablePlacement
        │  Collection       : Scorable, SortablePlacement
        │  MapCarousel      : Scorable, SortablePlacement
        │  AnnouncementColl.: Scorable, SortablePlacement
        │  StoreEntity      : Scorable, SortablePlacement  ← also horizontal
        └──────────────────────────────────────┘

        ┌─── Horizontal-only types implement Scorable only ───┐
        │  ItemStoreEntity  : Scorable
        │  DealStoreEntity  : Scorable
        └─────────────────────────────────────────────────────┘
```

No type is forced to implement an interface it doesn't need. Vertical types get positioning + scoring. Horizontal-only types get scoring alone.

### Which types implement Scorable?

| Type | Layer | Already has `id`? | Already has `predictionScore`? | Change needed |
|---|---|---|---|---|
| StoreCarousel | Vertical | Yes | Yes | Add `: Scorable`, `override`, `withPredictionScore` |
| ItemCarousel | Vertical | Yes | Yes | Same |
| DealCarousel | Vertical | Yes | Yes | Same |
| CollectionV2 | Vertical | Yes | Yes | Same |
| ItemCollection | Vertical | Yes | Yes | Same |
| Collection | Vertical | Yes | Yes | Same |
| MapCarousel | Vertical | Yes | Yes | Same |
| AnnouncementCollection | Vertical | Yes | Yes | Same |
| StoreEntity | Both | Yes (`Long`) | Yes | Same (id needs `toString()`) |
| ItemStoreEntity | Horizontal | Yes (`itemId`) | Yes | Same (id field name differs) |
| DealStoreEntity | Horizontal | Yes (`Long`) | Yes | Same |
| LiteStoreCollection | Vertical | Yes | **No** (uses maps) | Add `predictionScore` field, TBD |

11 of 12 types are a one-line change. LiteStoreCollection is the outlier.

---

## The Engine

Four components: `ScoreBoard`, `RankingStep`, `RankingHandler` (chain of responsibility), and `Ranker`. All are generic — the same engine runs both vertical and horizontal pipelines.

```kotlin
// ═══════════════════════════════════════════════════════════════
// 1. SCOREBOARD — external mutable score tracking
//
//    Domain objects are immutable data classes (val predictionScore).
//    The engine tracks mutable scores externally so data class
//    equals/hashCode/copy are never affected.
// ═══════════════════════════════════════════════════════════════

class ScoreBoard(items: List<Scorable>) {
    private val scores = HashMap<String, Double>(items.size)

    init {
        items.forEach { scores[it.id] = it.predictionScore ?: 0.0 }
    }

    operator fun get(id: String): Double = scores[id] ?: 0.0
    operator fun set(id: String, value: Double) { scores[id] = value }
}


// ═══════════════════════════════════════════════════════════════
// 2. RANKING STEP — domain logic contract
//
//    Steps are pure ranking logic. They score, reorder, or filter.
//    They don't know about chains, metrics, or conditions.
//    Generic on S (step type enum) — prevents mixing vertical
//    steps into a horizontal pipeline at compile time.
// ═══════════════════════════════════════════════════════════════

enum class VerticalStepType {
    RANK_ALL,             // Phase 1: wraps entire legacy pipeline
    // Phase 2: MODEL_SCORING, MULTIPLIER_BOOST, DIVERSITY_RERANK,
    //          POSITION_BOOSTING, FIXED_PINNING
}

enum class HorizontalStepType {
    RANK_ALL,             // Phase 1: wraps entire StoreCarouselService
    // Phase 2: MODEL_SCORING, SCORE_MODIFIER, CAMPAIGN_SORT,
    //          BUSINESS_RULES_SORT, ORDER_HISTORY_RERANK
}

sealed class StepParams
object RankAllParams : StepParams()

interface RankingStep<S : Enum<S>> {
    val stepType: S
    val paramsClass: Class<out StepParams>
    suspend fun execute(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
        params: StepParams,
    ): List<Scorable>
}

data class StepConfig<S : Enum<S>>(
    val type: S,
    val rawParams: String = "{}",   // Phase 1: always empty JSON
)


// ═══════════════════════════════════════════════════════════════
// 3. CHAIN OF RESPONSIBILITY — pipeline infrastructure
//
//    Steps are domain logic (servlet). Handlers are infrastructure
//    (servlet filter). The chain separates these concerns:
//
//      StepHandler        — wraps a RankingStep into the chain
//      MetricsHandler     — times its step, records duration
//      ConditionalHandler — skips step if experiment flag is off
//
//    Each link:  MetricsHandler(wraps StepHandler) → next link → ...
// ═══════════════════════════════════════════════════════════════

interface RankingHandler {
    suspend fun handle(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
    ): List<Scorable>
}

abstract class BaseHandler : RankingHandler {
    var next: RankingHandler? = null

    protected suspend fun next(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
    ): List<Scorable> = next?.handle(items, scoreBoard, context) ?: items
}

class StepHandler(
    private val step: RankingStep<*>,
    private val params: StepParams,
) : RankingHandler {
    override suspend fun handle(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
    ): List<Scorable> = step.execute(items, scoreBoard, context, params)
}

class MetricsHandler(
    private val stepName: String,
    private val metrics: MetricsClient,
    private val wrapped: RankingHandler,
) : BaseHandler() {
    override suspend fun handle(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
    ): List<Scorable> {
        val start = System.nanoTime()
        val result = wrapped.handle(items, scoreBoard, context)
        metrics.timer("ranking.step.$stepName.duration_ms",
            (System.nanoTime() - start) / 1_000_000)
        return next(result, scoreBoard, context)
    }
}

class ConditionalHandler(
    private val condition: (RankingContext) -> Boolean,
    private val wrapped: RankingHandler,
) : BaseHandler() {
    override suspend fun handle(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
    ): List<Scorable> {
        val result = if (condition(context)) {
            wrapped.handle(items, scoreBoard, context)
        } else {
            items
        }
        return next(result, scoreBoard, context)
    }
}


// ═══════════════════════════════════════════════════════════════
// 4. RANKER — assembles chain from config, executes it
// ═══════════════════════════════════════════════════════════════

class Ranker<S : Enum<S>>(
    private val stepRegistry: Map<S, RankingStep<S>>,
    private val metrics: MetricsClient,
) {
    suspend fun rank(
        items: List<Scorable>,
        pipeline: List<StepConfig<S>>,
        context: RankingContext,
    ): RankedResult {
        val scoreBoard = ScoreBoard(items)
        val chain = buildChain(pipeline, scoreBoard)
        val ranked = chain?.handle(items, scoreBoard, context) ?: items
        return RankedResult(ranked, scoreBoard)
    }

    private fun buildChain(
        pipeline: List<StepConfig<S>>,
        scoreBoard: ScoreBoard,
    ): RankingHandler? {
        val links: List<BaseHandler> = pipeline.mapNotNull { config ->
            val step = stepRegistry[config.type] ?: return@mapNotNull null
            val params = deserialize(config.rawParams, step.paramsClass)
            MetricsHandler(config.type.name, metrics, StepHandler(step, params))
        }
        for (i in 0 until links.size - 1) {
            links[i].next = links[i + 1]
        }
        return links.firstOrNull()
    }
}

data class RankedResult(
    val items: List<Scorable>,
    val scoreBoard: ScoreBoard,
)
```

---

## Putting It Together

### Domain changes (one-time, per type)

```kotlin
// ── Before ────────────────────────────────────────────────────
data class StoreCarousel(
    val id: String,
    val predictionScore: Double?,
    // ... 50+ other fields ...
) : Carousel, BaseCarousel, SortablePlacement

// ── After ─────────────────────────────────────────────────────
data class StoreCarousel(
    override val id: String,                  // add override
    override val predictionScore: Double?,    // add override
    // ... 50+ other fields (untouched) ...
) : Carousel, BaseCarousel, SortablePlacement, Scorable {
    override fun withPredictionScore(score: Double) = copy(predictionScore = score)
}

// Horizontal type — same pattern, Scorable only (no SortablePlacement)
data class ItemStoreEntity(
    override val id: String,
    override val predictionScore: Double?,
    // ...
) : BaseItemFeature, GroceryItemProvider, Scorable {
    override fun withPredictionScore(score: Double) = copy(predictionScore = score)
}
```

### Vertical ranking (Phase 1)

```kotlin
// ── Step: wraps the entire legacy pipeline ────────────────────
class VerticalRankAllStep(
    private val legacyRanker: BaseEntityRankerConfiguration,
) : RankingStep<VerticalStepType> {
    override val stepType = VerticalStepType.RANK_ALL
    override val paramsClass = RankAllParams::class.java

    override suspend fun execute(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
        params: StepParams,
    ): List<Scorable> {
        // items are still StoreCarousel, ItemCarousel, etc. at runtime.
        // Legacy ranker handles them as-is — nothing changes inside.
        val ranked = legacyRanker.rank(context, items)
        val result = ranked.filterIsInstance<Scorable>()
        result.forEach { scoreBoard[it.id] = it.predictionScore ?: 0.0 }
        return result
    }
}

// ── Wiring ────────────────────────────────────────────────────
@Bean fun verticalRanker(
    legacyRanker: BaseEntityRankerConfiguration,
    metrics: MetricsClient,
) = Ranker<VerticalStepType>(
    stepRegistry = mapOf(VerticalStepType.RANK_ALL to VerticalRankAllStep(legacyRanker)),
    metrics = metrics,
)

// ── Pipeline config ───────────────────────────────────────────
val verticalPipeline = listOf(StepConfig(type = VerticalStepType.RANK_ALL))

// ── Incision point (inside DefaultHomePagePostProcessor) ──────
suspend fun reOrderGlobalEntitiesV2(
    carousels: List<Any>,
    context: RankingContext,
    verticalRanker: Ranker<VerticalStepType>,
): List<Any> {
    val items = carousels.filterIsInstance<Scorable>()
    val (ranked, _) = verticalRanker.rank(items, verticalPipeline, context)
    return ranked  // correct order, correct scores
}
```

### Horizontal ranking (Phase 1) — same engine

```kotlin
// ── Step: wraps StoreCarouselService sort logic ───────────────
class HorizontalRankAllStep(
    private val storeCarouselService: StoreCarouselService,
) : RankingStep<HorizontalStepType> {
    override val stepType = HorizontalStepType.RANK_ALL
    override val paramsClass = RankAllParams::class.java

    override suspend fun execute(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
        params: StepParams,
    ): List<Scorable> {
        // items are still StoreEntity, ItemStoreEntity, etc. at runtime.
        val sorted = storeCarouselService.sortEntities(context, items)
        val result = sorted.filterIsInstance<Scorable>()
        result.forEach { scoreBoard[it.id] = it.predictionScore ?: 0.0 }
        return result
    }
}

// ── Wiring ────────────────────────────────────────────────────
@Bean fun horizontalRanker(
    storeCarouselService: StoreCarouselService,
    metrics: MetricsClient,
) = Ranker<HorizontalStepType>(
    stepRegistry = mapOf(HorizontalStepType.RANK_ALL to HorizontalRankAllStep(storeCarouselService)),
    metrics = metrics,
)

// ── Pipeline config ───────────────────────────────────────────
val horizontalPipeline = listOf(StepConfig(type = HorizontalStepType.RANK_ALL))

// ── Incision point (inside StoreCarousel assembly) ────────────
suspend fun rankStoresInCarousel(
    stores: List<StoreEntity>,
    context: RankingContext,
    horizontalRanker: Ranker<HorizontalStepType>,
): List<StoreEntity> {
    val items = stores.filterIsInstance<Scorable>()
    val (ranked, _) = horizontalRanker.rank(items, horizontalPipeline, context)
    return ranked.filterIsInstance<StoreEntity>()
}
```

**Same `Ranker`. Same `ScoreBoard`. Same `RankingHandler` chain.** The only differences are the step type enum (`VerticalStepType` vs `HorizontalStepType`) and the step implementation. The type system prevents accidentally passing a vertical step to a horizontal pipeline.

### Execution trace (vertical, Phase 1)

```
reOrderGlobalEntitiesV2(carousels, context)
│
├─ items = carousels.filterIsInstance<Scorable>()
│    └─ [StoreCarousel, ItemCarousel, DealCarousel, StoreEntity, ...]
│
├─ Ranker.rank(items, pipeline, context)
│  ├─ ScoreBoard initialized from items' predictionScores
│  ├─ chain = MetricsHandler("RANK_ALL", StepHandler(VerticalRankAllStep))
│  │
│  └─ chain.handle()
│     ├─ MetricsHandler: start timer
│     ├─ StepHandler → VerticalRankAllStep.execute()
│     │  ├─ legacyRanker.rank(context, items)   ← same code as today
│     │  ├─ ScoreBoard updated with result scores
│     │  └─ return ranked items
│     ├─ MetricsHandler: record 127ms
│     └─ next → null (end of chain)
│
└─ return ranked items
```

---

## Extending to the Future

### Phase 2: Granular steps

Break `RANK_ALL` into individual steps. The engine doesn't change — only the step registry and pipeline config.

```kotlin
enum class VerticalStepType {
    RANK_ALL,              // Phase 1 (kept for rollback)
    MODEL_SCORING,         // wraps EntityScorer.score()
    MULTIPLIER_BOOST,      // wraps BlendingUtil.blendBundle()
    DIVERSITY_RERANK,      // wraps BlendingUtil.rerankWithDiversity()
    POSITION_BOOSTING,     // wraps BoostingBundle.boosted()
    FIXED_PINNING,         // wraps pin/flow separation
}

class ModelScoringStep(
    private val entityScorer: EntityScorer,
) : RankingStep<VerticalStepType> {
    override val stepType = VerticalStepType.MODEL_SCORING
    override val paramsClass = ModelScoringParams::class.java

    override suspend fun execute(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
        params: StepParams,
    ): List<Scorable> {
        val scoreBundle = entityScorer.score(context, items)
        scoreBundle.forEach { (id, score) -> scoreBoard[id] = score }
        return items   // scoring doesn't reorder
    }
}

// Pipeline changes. Engine stays identical.
val granularPipeline = listOf(
    StepConfig(type = VerticalStepType.MODEL_SCORING),
    StepConfig(type = VerticalStepType.MULTIPLIER_BOOST),
    StepConfig(type = VerticalStepType.DIVERSITY_RERANK),
    StepConfig(type = VerticalStepType.POSITION_BOOSTING),
    StepConfig(type = VerticalStepType.FIXED_PINNING),
)

// Incision point: only apply-back changes
val (ranked, scores) = verticalRanker.rank(items, granularPipeline, context)
return ranked.map { it.withPredictionScore(scores[it.id]) }
```

Chain becomes:

```
MetricsHandler("MODEL_SCORING") → MetricsHandler("MULTIPLIER_BOOST") → MetricsHandler("DIVERSITY_RERANK") → ...
     ↓ wraps                          ↓ wraps                               ↓ wraps
   StepHandler                      StepHandler                           StepHandler
     ↓ executes                       ↓ executes                            ↓ executes
   ModelScoringStep                 MultiplierBoostStep                   DiversityRerankStep
```

### Phase 2: Experiment-gated steps

Use `ConditionalHandler` to skip steps based on experiment flags. No changes to the step itself.

```kotlin
val conditions = mapOf(
    VerticalStepType.DIVERSITY_RERANK to { ctx: RankingContext ->
        ctx.hasExperiment("diversity_v2_enabled")
    },
)

// Chain builder wraps conditional steps automatically
fun <S : Enum<S>> Ranker<S>.buildChainWithConditions(
    pipeline: List<StepConfig<S>>,
    conditions: Map<S, (RankingContext) -> Boolean>,
): RankingHandler? {
    val links = pipeline.mapNotNull { config ->
        val step = stepRegistry[config.type] ?: return@mapNotNull null
        val params = deserialize(config.rawParams, step.paramsClass)

        var handler: RankingHandler = StepHandler(step, params)
        conditions[config.type]?.let { condition ->
            handler = ConditionalHandler(condition, handler)
        }
        MetricsHandler(config.type.name, metrics, handler)
    }
    links.zipWithNext().forEach { (a, b) -> a.next = b }
    return links.firstOrNull()
}
```

### Phase 2: Shadow mode

Compare legacy vs new ranking path. Return legacy result (safe), emit comparison metrics.

```kotlin
class ShadowHandler(
    private val legacy: RankingHandler,
    private val candidate: RankingHandler,
    private val metrics: MetricsClient,
) : BaseHandler() {
    override suspend fun handle(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
    ): List<Scorable> {
        val legacyBoard = ScoreBoard(items)
        val candidateBoard = ScoreBoard(items)

        val legacyResult = legacy.handle(items, legacyBoard, context)
        val candidateResult = candidate.handle(items, candidateBoard, context)

        // Compare
        val orderMatch = legacyResult.map { it.id } == candidateResult.map { it.id }
        metrics.gauge("ranking.shadow.order_match", if (orderMatch) 1.0 else 0.0)

        // Return legacy (safe)
        return next(legacyResult, scoreBoard, context)
    }
}
```

### Phase 2: Typed metadata for inter-step communication

When granular steps need to pass data between each other (e.g., calibration multiplier from scoring → boost step):

```kotlin
class MetadataKey<T>(val name: String)

object Keys {
    val CALIBRATION_MULTIPLIER = MetadataKey<Double>("calibrationMultiplier")
    val VERTICAL_INTENT = MetadataKey<Double>("verticalIntentMultiplier")
}

// ScoreBoard extended with metadata
class ScoreBoard(items: List<Scorable>) {
    private val scores = HashMap<String, Double>(items.size)
    private val metadata = HashMap<String, MutableMap<String, Any>>()

    // ... score methods ...

    @Suppress("UNCHECKED_CAST")
    fun <T> getMetadata(id: String, key: MetadataKey<T>): T? =
        metadata[id]?.get(key.name) as? T

    fun <T : Any> setMetadata(id: String, key: MetadataKey<T>, value: T) {
        metadata.getOrPut(id) { mutableMapOf() }[key.name] = value
    }
}
```

### Phase 2: StepParams with real config

When experiments need to override step config (e.g., different model for scoring):

```kotlin
data class ModelScoringParams(
    val modelId: String = "default_sibyl_v3",
    val calibrationEnabled: Boolean = true,
) : StepParams()

// Pipeline config from experiment resolution
val experimentPipeline = listOf(
    StepConfig(
        type = VerticalStepType.MODEL_SCORING,
        rawParams = """{"modelId": "sibyl_v4_candidate", "calibrationEnabled": false}""",
    ),
    StepConfig(type = VerticalStepType.MULTIPLIER_BOOST),
    StepConfig(type = VerticalStepType.DIVERSITY_RERANK),
)
```

### Phase 3: DAG with parallel branches

When parallel execution matters (e.g., scoring with multiple models concurrently):

```
Phase 1:  RANK_ALL

Phase 2:  MODEL_SCORING → MULTIPLIER_BOOST → DIVERSITY_RERANK → POSITION_BOOSTING → FIXED_PINNING

Phase 3:                ┌→ MODEL_A_SCORING ─┐
          FEATURE_PREP ─┤                    ├→ SCORE_MERGE → DIVERSITY_RERANK → PINNING
                        └→ MODEL_B_SCORING ─┘
```

Handler interface stays identical. Execution engine adds fan-out/fan-in:
- Nodes declare dependencies (DAG edges)
- Engine does topological sort
- Independent nodes execute concurrently (coroutines)
- Merge nodes wait for all inputs

---

## Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Inheritance vs composition | Inheritance (`Scorable` interface) | All 12 types already have `id` + `predictionScore`. One-line change. Composition reconstructs the same capability with more boilerplate. |
| Scorable vs SortablePlacement | Orthogonal — not a hierarchy | Scorable = scoring capability. SortablePlacement = vertical positioning. Vertical types implement both. Horizontal types implement only Scorable. |
| Mutable scores | External `ScoreBoard` | Data classes stay immutable. No `var score` breaking `equals`/`hashCode`/`copy`. |
| Apply-back | `withPredictionScore()` on each type | Polymorphic — eliminates `when` blocks. Each type implements via `copy()`. |
| Pipeline execution | Chain of Responsibility | Cross-cutting concerns (metrics, conditions, shadow mode) are composable handlers, not inline noise. |
| Step type generics | `S : Enum<S>` per layer | `Ranker<VerticalStepType>` can't accept horizontal steps. Compiler prevents it. |
| Phase 1 granularity | One `RANK_ALL` step | Wraps legacy pipeline as-is. No bridge conversions. Granular steps in Phase 2. |
| StepParams Phase 1 | Empty stubs | Existing methods read their own config. Real params for experiment-driven config in Phase 2. |

## Open Questions

- **LiteStoreCollection**: Uses `storeScoreMap` instead of `predictionScore`. Add a `predictionScore` field? Compute from map? Exclude from Scorable?
- **StoreEntity `id` type**: StoreEntity has `id: Long`, Scorable expects `String`. Use `id.toString()` or change Scorable to `Any`?
- **Chain lifecycle**: Build once at startup and reuse, or rebuild per-request from pipeline config?
