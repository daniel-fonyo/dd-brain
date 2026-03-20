# POC: Scorable + ScoreBoard + Chain of Responsibility

> Domain objects implement `Scorable` directly (one-line change per class). No adapters. Mutable scores in external `ScoreBoard`. Pipeline executed via Chain of Responsibility — cross-cutting concerns (metrics, conditional execution, shadow mode) are first-class handlers. Phase 1: one coarse `RANK_ALL` handler wrapping the legacy pipeline.

## Design Decisions Captured

| Decision | Choice | Why |
| --- | --- | --- |
| Adapters vs direct interface | Domain objects implement `Scorable` directly | Eliminates 9 adapter wrapper classes. One-line change per data class. |
| Mutable score tracking | External `ScoreBoard` in the engine | Data classes stay immutable. No `var score` breaking `equals`/`hashCode`/`copy`. |
| Apply-back mechanism | `withPredictionScore(score): Scorable` on each type | Each data class implements via `copy(predictionScore = score)`. Polymorphic — no `when` block. |
| Pipeline execution | Chain of Responsibility | Cross-cutting concerns (metrics, conditional skip, shadow mode) are handlers, not `try/catch` noise in a for-loop. Each handler decides to process, skip, or fork. |
| Generics | `Ranker<S : Enum<S>>`, `RankingStep<S : Enum<S>>` | One engine, one step interface. Type system prevents mixing vertical/horizontal steps. |
| Typed metadata keys | `MetadataKey<T>` | Compile-time safety. Unused in Phase 1, ready for inter-step communication. |
| StepParams in Phase 1 | Empty stubs — no data, no logic | Existing methods read their own config internally. Real params are a future concern. |
| Phase 1 granularity | One coarse `RANK_ALL` step | Wraps `BaseEntityRankerConfiguration.rank()` as-is. No bridge conversions. |

---

## Full Example Code

```kotlin
// ═══════════════════════════════════════════════════════════════
// 1. CORE ABSTRACTIONS — shared between vertical & horizontal
// ═══════════════════════════════════════════════════════════════

// ── Scorable interface ────────────────────────────────────────
// Minimal contract. Domain objects already have id + predictionScore.
// Adding this interface to each data class is a one-line change.

interface Scorable {
    val id: String
    val predictionScore: Double?
    fun withPredictionScore(score: Double): Scorable
}


// ── ScoreBoard — external mutable score tracking ─────────────
// Domain objects stay immutable (val predictionScore).
// The engine tracks mutable scores here instead.

class ScoreBoard(items: List<Scorable>) {
    private val scores = HashMap<String, Double>(items.size)

    init {
        items.forEach { scores[it.id] = it.predictionScore ?: 0.0 }
    }

    operator fun get(id: String): Double = scores[id] ?: 0.0
    operator fun set(id: String, value: Double) { scores[id] = value }
}


// ── Typed metadata keys ──────────────────────────────────────
// Phase 1: unused. Phase 2: inter-step communication.

class MetadataKey<T>(val name: String)

@Suppress("UNCHECKED_CAST")
operator fun <T> MutableMap<String, Any>.get(key: MetadataKey<T>): T? = this[key.name] as? T
operator fun <T : Any> MutableMap<String, Any>.set(key: MetadataKey<T>, value: T) { this[key.name] = value }


// ── Step type enums ──────────────────────────────────────────

enum class VerticalStepType {
    RANK_ALL,             // Phase 1 — wraps entire BaseEntityRankerConfiguration.rank()
    // ── Phase 2: granular breakdown ──
    // MODEL_SCORING,
    // MULTIPLIER_BOOST,
    // DIVERSITY_RERANK,
    // POSITION_BOOSTING,
    // FIXED_PINNING,
}

enum class HorizontalStepType {
    RANK_ALL,             // Phase 1 — wraps entire StoreRanker/CampaignRanker
    // ── Phase 2: granular breakdown ──
    // MODEL_SCORING,
    // SCORE_MODIFIER,
    // CAMPAIGN_SORT,
    // BUSINESS_RULES_SORT,
    // ORDER_HISTORY_RERANK,
}


// ── Step params ──────────────────────────────────────────────

sealed class StepParams
object RankAllParams : StepParams()


// ═══════════════════════════════════════════════════════════════
// 2. RANKING STEP — the domain logic contract
//
//    Steps are pure domain logic. They don't know about chains,
//    metrics, or conditions. They just score/reorder items.
//    Think: servlet vs servlet filter. Step = servlet.
// ═══════════════════════════════════════════════════════════════

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
    val rawParams: String = "{}",
)


// ═══════════════════════════════════════════════════════════════
// 3. CHAIN OF RESPONSIBILITY — pipeline infrastructure
//
//    Handlers form a linked chain. Each handler:
//      - Processes the request (or delegates to a wrapped handler)
//      - Passes to the next handler in the chain
//
//    Separation of concerns:
//      RankingStep  = domain logic (scoring, reranking)
//      StepHandler  = wraps a step into the chain
//      MetricsHandler = times the step, records metrics
//      ConditionalHandler = skips based on context/experiments
//
//    Chain structure (each "link" is a decorated step):
//      MetricsHandler → MetricsHandler → MetricsHandler → ∅
//        ↓ wraps           ↓ wraps           ↓ wraps
//      StepHandler       StepHandler       StepHandler
//        ↓ executes        ↓ executes        ↓ executes
//      RankAllStep       ModelScoring      DiversityRerank
// ═══════════════════════════════════════════════════════════════

// ── Handler interface ─────────────────────────────────────────

interface RankingHandler {
    suspend fun handle(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
    ): List<Scorable>
}

// Base class with next-chaining
abstract class BaseHandler : RankingHandler {
    var next: RankingHandler? = null

    protected suspend fun next(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
    ): List<Scorable> = next?.handle(items, scoreBoard, context) ?: items
}


// ── StepHandler — wraps a RankingStep into the chain ──────────
// Leaf handler. Executes the step, does NOT chain internally.
// Chaining is handled by the outer decorator (MetricsHandler).

class StepHandler(
    private val step: RankingStep<*>,
    private val params: StepParams,
) : RankingHandler {
    override suspend fun handle(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
    ): List<Scorable> {
        return step.execute(items, scoreBoard, context, params)
    }
}


// ── MetricsHandler — times the wrapped step, chains to next ──
// Decorates a StepHandler. Measures ONLY its step's duration
// (not the rest of the chain), then passes to next link.

class MetricsHandler(
    private val stepName: String,
    private val metrics: MetricsClient,
    private val wrapped: RankingHandler,  // the StepHandler
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


// ── ConditionalHandler — skips step based on context ──────────
// Wraps a handler. If condition is false, skips entirely and
// passes items unchanged to next link. Useful for experiment
// flags, feature gates, A/B variants.

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
            items  // skip — condition not met
        }
        return next(result, scoreBoard, context)
    }
}


// ── Chain builder — assembles pipeline config into a chain ────
// Each StepConfig becomes: MetricsHandler(StepHandler(RankingStep))
// Links are chained: handler1.next = handler2.next = handler3...

fun <S : Enum<S>> buildChain(
    pipeline: List<StepConfig<S>>,
    stepRegistry: Map<S, RankingStep<S>>,
    metrics: MetricsClient,
): RankingHandler? {
    val links: List<BaseHandler> = pipeline.mapNotNull { config ->
        val step = stepRegistry[config.type] ?: return@mapNotNull null
        val params = deserialize(config.rawParams, step.paramsClass)

        // Build from inside out: step → metrics
        val stepHandler = StepHandler(step, params)
        MetricsHandler(config.type.name, metrics, stepHandler)
    }

    // Wire the chain: each link's next → following link
    for (i in 0 until links.size - 1) {
        links[i].next = links[i + 1]
    }

    return links.firstOrNull()
}


// ═══════════════════════════════════════════════════════════════
// 4. RANKER — builds chain, executes it, returns result
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

        val chain = buildChain(pipeline, stepRegistry, metrics)
        val ranked = chain?.handle(items, scoreBoard, context) ?: items

        return RankedResult(ranked, scoreBoard)
    }
}

data class RankedResult(
    val items: List<Scorable>,
    val scoreBoard: ScoreBoard,
)


// ═══════════════════════════════════════════════════════════════
// 5. DOMAIN CHANGES — one-line per type
//
//    Each data class already has `id` and `predictionScore`.
//    Changes: add `: Scorable`, add `override`, add withPredictionScore.
//    No new classes. No wrappers.
//
//    Vertical (9): StoreCarousel, ItemCarousel, DealCarousel,
//      LiteStoreCollection, CollectionV2, ItemCollection,
//      MapCarousel, AnnouncementCollection, StoreEntity
//    Horizontal (3): StoreEntity, DealStoreEntity, ItemStoreEntity
// ═══════════════════════════════════════════════════════════════

// ── Before ────────────────────────────────────────────────────
data class StoreCarousel(
    val id: String,
    // ... 50+ other fields ...
    val predictionScore: Double?,
    val scoreMultiplier: Double?,
    val calibrationMultiplier: Double?,
    // ...
) : Carousel, BaseCarousel, SortablePlacement

// ── After (3 changes highlighted) ─────────────────────────────
data class StoreCarousel(
    override val id: String,                    // ← add override
    // ... 50+ other fields (untouched) ...
    override val predictionScore: Double?,      // ← add override
    val scoreMultiplier: Double?,
    val calibrationMultiplier: Double?,
    // ...
) : Carousel, BaseCarousel, SortablePlacement, Scorable {  // ← add Scorable
    override fun withPredictionScore(score: Double) = copy(predictionScore = score)
}

// Same pattern for all other types:
data class ItemCarousel(
    override val id: String,
    override val predictionScore: Double?,
    // ...
) : Carousel, BaseCarousel, SortablePlacement, Scorable {
    override fun withPredictionScore(score: Double) = copy(predictionScore = score)
}

data class DealCarousel(
    override val id: String,
    override val predictionScore: Double?,
    // ...
) : Carousel, BaseCarousel, Scorable {
    override fun withPredictionScore(score: Double) = copy(predictionScore = score)
}

// LiteStoreCollection — outlier. Uses maps instead of single
// predictionScore. Add predictionScore field or compute from map. TBD.
data class LiteStoreCollection(
    override val id: String,
    override val predictionScore: Double? = null,
    val storeScoreMap: Map<String, Double> = emptyMap(),
    // ...
) : Scorable {
    override fun withPredictionScore(score: Double) = copy(predictionScore = score)
}


// ═══════════════════════════════════════════════════════════════
// 6. PHASE 1 STEP — RANK_ALL
//
//    Wraps the ENTIRE existing pipeline as one opaque step.
//    No bridge conversions. No new scoring logic.
// ═══════════════════════════════════════════════════════════════

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
        // Legacy ranker handles them as-is — same types, same method.
        val ranked: List<Any> = legacyRanker.rank(context, items)

        // Legacy ranker returns copies with updated predictionScore.
        // Capture in ScoreBoard for apply-back.
        val result = ranked.filterIsInstance<Scorable>()
        result.forEach { scoreBoard[it.id] = it.predictionScore ?: 0.0 }

        return result
    }
}


// ═══════════════════════════════════════════════════════════════
// 7. SPRING WIRING
// ═══════════════════════════════════════════════════════════════

@Configuration
class UbpConfig {
    @Bean
    fun verticalStepRegistry(
        legacyRanker: BaseEntityRankerConfiguration,
    ): Map<VerticalStepType, RankingStep<VerticalStepType>> = mapOf(
        VerticalStepType.RANK_ALL to VerticalRankAllStep(legacyRanker),
    )

    @Bean
    fun verticalRanker(
        registry: Map<VerticalStepType, RankingStep<VerticalStepType>>,
        metrics: MetricsClient,
    ): Ranker<VerticalStepType> = Ranker(registry, metrics)
}


// ═══════════════════════════════════════════════════════════════
// 8. PIPELINE CONFIG
//    Phase 1: single RANK_ALL step.
//    Production: comes from upstream hash-based traffic bucketing.
//    See: experiment-traffic-industry-research.md
// ═══════════════════════════════════════════════════════════════

val verticalPipeline = listOf(
    StepConfig(type = VerticalStepType.RANK_ALL),
)


// ═══════════════════════════════════════════════════════════════
// 9. THE INCISION POINT — inside DefaultHomePagePostProcessor
//
//    No adapt `when` block. No apply-back `when` block.
//    No adapter classes. Items in → chain executes → items out.
// ═══════════════════════════════════════════════════════════════

suspend fun reOrderGlobalEntitiesV2(
    carousels: List<Any>,
    context: RankingContext,
    verticalRanker: Ranker<VerticalStepType>,
    pipeline: List<StepConfig<VerticalStepType>>,
): List<Any> {

    // Domain objects already implement Scorable — no conversion.
    val items: List<Scorable> = carousels.filterIsInstance<Scorable>()

    // Chain executes: MetricsHandler → StepHandler(RANK_ALL) → ∅
    val (ranked, scores) = verticalRanker.rank(items, pipeline, context)

    // Phase 1: RANK_ALL already produced copies with correct scores.
    return ranked

    // Phase 2: apply scoreBoard back to items.
    // return ranked.map { it.withPredictionScore(scores[it.id]) }
}
```

---

## Chain Execution Trace — Phase 1

```
reOrderGlobalEntitiesV2(carousels, context)
│
├─ items = carousels.filterIsInstance<Scorable>()
├─ scoreBoard = ScoreBoard(items)
│
├─ chain = MetricsHandler("RANK_ALL", wrapped=StepHandler(VerticalRankAllStep))
│
└─ chain.handle(items, scoreBoard, context)
   │
   ├─ MetricsHandler.handle()
   │  ├─ start timer
   │  ├─ wrapped.handle() → StepHandler.handle()
   │  │  └─ VerticalRankAllStep.execute()
   │  │     ├─ legacyRanker.rank(context, items)  ← same as today
   │  │     ├─ scoreBoard updated with result scores
   │  │     └─ return ranked items
   │  ├─ record duration metric
   │  └─ next() → null (end of chain)
   │
   └─ return ranked items with correct scores and order
```

## Chain Execution Trace — Phase 2 (Granular)

```
chain = MetricsHandler("MODEL_SCORING") → MetricsHandler("MULTIPLIER_BOOST") → MetricsHandler("DIVERSITY_RERANK") → ∅
          ↓ wraps                            ↓ wraps                               ↓ wraps
        StepHandler(ModelScoringStep)      StepHandler(MultiplierBoostStep)       StepHandler(DiversityRerankStep)

chain.handle(items, scoreBoard, context)
│
├─ MetricsHandler("MODEL_SCORING").handle()
│  ├─ ⏱ start
│  ├─ ModelScoringStep.execute() → scores in ScoreBoard
│  ├─ ⏱ record 45ms
│  └─ next() →
│
├─ MetricsHandler("MULTIPLIER_BOOST").handle()
│  ├─ ⏱ start
│  ├─ MultiplierBoostStep.execute() → scores adjusted in ScoreBoard
│  ├─ ⏱ record 12ms
│  └─ next() →
│
├─ MetricsHandler("DIVERSITY_RERANK").handle()
│  ├─ ⏱ start
│  ├─ DiversityRerankStep.execute() → items reordered
│  ├─ ⏱ record 8ms
│  └─ next() → null (end)
│
└─ return reordered items
```

## Chain Composition Examples

```kotlin
// ── Example 1: Conditional step (experiment-gated) ────────────
// Diversity rerank only runs if experiment flag is on.
// ConditionalHandler wraps the step; if condition fails, skips.

fun <S : Enum<S>> buildChainWithConditions(
    pipeline: List<StepConfig<S>>,
    stepRegistry: Map<S, RankingStep<S>>,
    metrics: MetricsClient,
    conditions: Map<S, (RankingContext) -> Boolean> = emptyMap(),
): RankingHandler? {
    val links: List<BaseHandler> = pipeline.mapNotNull { config ->
        val step = stepRegistry[config.type] ?: return@mapNotNull null
        val params = deserialize(config.rawParams, step.paramsClass)

        // Inside out: step → conditional (optional) → metrics
        var handler: RankingHandler = StepHandler(step, params)

        val condition = conditions[config.type]
        if (condition != null) {
            val conditional = ConditionalHandler(condition, handler)
            handler = conditional
        }

        MetricsHandler(config.type.name, metrics, handler)
    }

    for (i in 0 until links.size - 1) {
        links[i].next = links[i + 1]
    }
    return links.firstOrNull()
}

// Usage:
val conditions = mapOf(
    VerticalStepType.DIVERSITY_RERANK to { ctx: RankingContext ->
        ctx.hasExperiment("diversity_v2_enabled")
    },
)
val chain = buildChainWithConditions(pipeline, registry, metrics, conditions)


// ── Example 2: Shadow mode handler ───────────────────────────
// Runs both legacy and new path, compares results, emits metrics.
// Callers see only the legacy result (safe rollout).

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
        // Run both paths
        val legacyBoard = ScoreBoard(items)
        val candidateBoard = ScoreBoard(items)

        val legacyResult = legacy.handle(items, legacyBoard, context)
        val candidateResult = candidate.handle(items, candidateBoard, context)

        // Compare and emit metrics
        val orderMatch = legacyResult.map { it.id } == candidateResult.map { it.id }
        metrics.gauge("ranking.shadow.order_match", if (orderMatch) 1.0 else 0.0)

        // Return legacy result (safe — candidate is shadow only)
        return next(legacyResult, legacyBoard, context)
    }
}
```

---

## Phase 2 Sketch — Granular Steps

```kotlin
enum class VerticalStepType {
    RANK_ALL,
    MODEL_SCORING,
    MULTIPLIER_BOOST,
    DIVERSITY_RERANK,
    POSITION_BOOSTING,
    FIXED_PINNING,
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
        // Items are still original types at runtime.
        val scoreBundle = entityScorer.score(context, items)
        scoreBundle.forEach { (id, score) -> scoreBoard[id] = score }
        return items  // scoring doesn't reorder
    }
}

// Phase 2 pipeline
val verticalPipelineV2 = listOf(
    StepConfig(type = VerticalStepType.MODEL_SCORING),
    StepConfig(type = VerticalStepType.MULTIPLIER_BOOST),
    StepConfig(type = VerticalStepType.DIVERSITY_RERANK),
    StepConfig(type = VerticalStepType.POSITION_BOOSTING),
    StepConfig(type = VerticalStepType.FIXED_PINNING),
)
```

## Phase 3 Sketch — DAG Extension

Phase 1-2 pipelines are linear chains. When parallel branches matter
(e.g., score via model A and model B concurrently, then merge), the
chain extends to a DAG:

```
Phase 1:  RANK_ALL

Phase 2:  MODEL_SCORING → MULTIPLIER_BOOST → DIVERSITY_RERANK → POSITION_BOOSTING → FIXED_PINNING

Phase 3:                ┌→ MODEL_A_SCORING ─┐
          FEATURE_PREP ─┤                    ├→ SCORE_MERGE → DIVERSITY_RERANK → PINNING
                        └→ MODEL_B_SCORING ─┘
```

The handler interface stays identical. What changes is the execution engine:
- Nodes declare their dependencies (edges in the DAG)
- Engine does topological sort
- Nodes with satisfied dependencies execute concurrently (coroutines)
- Merge nodes wait for all inputs

CoR is the natural foundation — DAG nodes ARE handlers, the wiring just
allows fan-out/fan-in instead of strictly linear next pointers.

---

## What Changed from Previous POC

| # | Previous (Scorable + for-loop) | Current (Scorable + CoR) | Why |
|---|---|---|---|
| 1 | `for (stepConfig in pipeline) { step.execute() }` | `chain.handle()` dispatches through linked handlers | Cross-cutting concerns are handlers, not inline noise. |
| 2 | Metrics would be `try/finally` blocks in the loop | `MetricsHandler` wraps each step, measures only its duration | Clean separation. Add/remove metrics without touching steps. |
| 3 | Conditional execution = `if` in the loop | `ConditionalHandler` wraps a step, skips if condition fails | Conditions are composable. Experiment flags become handlers. |
| 4 | Shadow mode = ??? | `ShadowHandler` forks: runs both paths, compares, returns legacy | Shadow mode is a handler — no special engine support needed. |
| 5 | No execution visibility | Chain structure is inspectable and loggable | Can log "which handlers ran" for debugging/audit. |

## Phase 1 vs Future

| Concern | Phase 1 | Future |
|---|---|---|
| **Chain length** | 1 link: `MetricsHandler(RANK_ALL)` | 5+ links with conditional/shadow handlers |
| **Step granularity** | One `RANK_ALL` wrapping legacy pipeline | Granular steps: scoring, boosting, reranking, pinning |
| **Cross-cutting** | MetricsHandler per step | + ConditionalHandler, ShadowHandler, LoggingHandler |
| **ScoreBoard** | Captures legacy output for bookkeeping | Primary score-tracking — steps read/write it |
| **StepParams** | Empty stubs | Real config. Experiments override via JSON. |
| **Pipeline shape** | Linear (1 handler) | Linear chain → DAG with parallel branches |

## Open Questions

- **LiteStoreCollection**: Uses `storeScoreMap` instead of `predictionScore`. Add field? Compute from map? Exclude from Scorable?
- **withPredictionScore return type**: Returns `Scorable` not concrete type. Downstream must cast. Acceptable? (Same as today — ranking returns `List<Any>`.)
- **Handler ordering in buildChain**: Current approach: metrics wraps step, links chain linearly. Should this be configurable per-step (e.g., some steps get conditional + metrics, others just metrics)?
- **Chain immutability**: Current chain uses mutable `var next`. Should chain be rebuilt per-request (from pipeline config), or built once at startup and reused?
