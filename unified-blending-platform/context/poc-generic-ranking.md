# Ranking Abstraction POC

## The Problem

9 vertical placement types and 3+ horizontal entity types. All have `id` and `predictionScore` but no shared interface. Ranking logic scattered across `BaseEntityRankerConfiguration`, `StoreCarouselService`, and a dozen utility classes. No uniform way to score, reorder, or experiment.

## Scorable Interface

```kotlin
interface Scorable {
    val id: String
    val predictionScore: Double?
    fun withPredictionScore(score: Double): Scorable
}
```

Every rankable type already has `id` and `predictionScore`. Implementing `Scorable` = adding `override` to existing fields + a one-line `withPredictionScore` via `copy()`.

**Why inheritance over composition?** The fields already exist. We're formalizing a convention into a compile-time contract. Composition (`Ranked<T>` wrappers or `ScoreExtractor<T>` registries) reconstructs the same capability with more boilerplate and reintroduces `when` blocks.

### Scorable vs SortablePlacement

`SortablePlacement` handles vertical positioning. `Scorable` handles scoring. Orthogonal capabilities:

```
Scorable                          SortablePlacement
"I can be scored"                 "I can be positioned vertically"

  ┌─── Vertical types implement BOTH ───┐
  │  StoreCarousel    : Scorable, SortablePlacement
  │  ItemCarousel     : Scorable, SortablePlacement
  │  DealCarousel     : Scorable, SortablePlacement
  │  CollectionV2     : Scorable, SortablePlacement
  │  ItemCollection   : Scorable, SortablePlacement
  │  Collection       : Scorable, SortablePlacement
  │  MapCarousel      : Scorable, SortablePlacement
  │  Announcement...  : Scorable, SortablePlacement
  │  StoreEntity      : Scorable, SortablePlacement  ← also horizontal
  └──────────────────────────────────────┘

  ┌─── Horizontal-only: Scorable only ──┐
  │  ItemStoreEntity  : Scorable
  │  DealStoreEntity  : Scorable
  └─────────────────────────────────────┘
```

11 of 12 types: one-line change. LiteStoreCollection is the outlier (uses maps instead of `predictionScore`).

---

## The Engine

Steps are domain logic. Handlers are infrastructure (metrics, conditions). Chain of Responsibility links them. Ranker assembles and executes the chain. All generic — same engine for vertical and horizontal.

```kotlin
// ── Step type enums (compile-time layer safety) ───────────────

enum class VerticalStepType { RANK_ALL }

enum class HorizontalStepType { RANK_ALL }

// ── Step: domain logic contract ───────────────────────────────
// Items in → items out. Pure function. No mutable state.

interface RankingStep<S : Enum<S>> {
    val stepType: S
    suspend fun execute(items: List<Scorable>, context: RankingContext): List<Scorable>
}

// ── Chain of Responsibility: infrastructure ───────────────────

interface RankingHandler {
    suspend fun handle(items: List<Scorable>, context: RankingContext): List<Scorable>
}

abstract class BaseHandler : RankingHandler {
    var next: RankingHandler? = null
    protected suspend fun next(items: List<Scorable>, ctx: RankingContext) =
        next?.handle(items, ctx) ?: items
}

class StepHandler(private val step: RankingStep<*>) : RankingHandler {
    override suspend fun handle(items: List<Scorable>, ctx: RankingContext) =
        step.execute(items, ctx)
}

class MetricsHandler(
    private val name: String,
    private val metrics: MetricsClient,
    private val wrapped: RankingHandler,
) : BaseHandler() {
    override suspend fun handle(items: List<Scorable>, ctx: RankingContext): List<Scorable> {
        val start = System.nanoTime()
        val result = wrapped.handle(items, ctx)
        metrics.timer("ranking.step.$name.duration_ms", (System.nanoTime() - start) / 1_000_000)
        return next(result, ctx)
    }
}

// ── Ranker: assembles chain, executes it ──────────────────────

class Ranker<S : Enum<S>>(
    private val stepRegistry: Map<S, RankingStep<S>>,
    private val metrics: MetricsClient,
) {
    suspend fun rank(items: List<Scorable>, pipeline: List<S>, ctx: RankingContext): List<Scorable> {
        val links = pipeline.mapNotNull { type ->
            stepRegistry[type]?.let { MetricsHandler(type.name, metrics, StepHandler(it)) }
        }
        for (i in 0 until links.size - 1) links[i].next = links[i + 1]
        return links.firstOrNull()?.handle(items, ctx) ?: items
    }
}
```

---

## Phase 1: Vertical Ranking

```kotlin
// ── Domain change (repeat for all 9 types) ────────────────────

// Before:
data class StoreCarousel(
    val id: String,
    val predictionScore: Double?,
    // ... 50+ other fields ...
) : Carousel, BaseCarousel, SortablePlacement

// After:
data class StoreCarousel(
    override val id: String,
    override val predictionScore: Double?,
    // ... 50+ other fields (untouched) ...
) : Carousel, BaseCarousel, SortablePlacement, Scorable {
    override fun withPredictionScore(score: Double) = copy(predictionScore = score)
}


// ── Step: wraps entire legacy pipeline ────────────────────────

class VerticalRankAllStep(
    private val legacyRanker: BaseEntityRankerConfiguration,
) : RankingStep<VerticalStepType> {
    override val stepType = VerticalStepType.RANK_ALL

    override suspend fun execute(items: List<Scorable>, ctx: RankingContext): List<Scorable> {
        // items are still StoreCarousel, ItemCarousel, etc. at runtime
        val ranked = legacyRanker.rank(ctx, items)
        return ranked.filterIsInstance<Scorable>()
    }
}


// ── Wiring + incision point ───────────────────────────────────

@Bean fun verticalRanker(legacy: BaseEntityRankerConfiguration, metrics: MetricsClient) =
    Ranker<VerticalStepType>(mapOf(VerticalStepType.RANK_ALL to VerticalRankAllStep(legacy)), metrics)

suspend fun reOrderGlobalEntitiesV2(
    carousels: List<Any>,
    ctx: RankingContext,
    ranker: Ranker<VerticalStepType>,
): List<Any> {
    val items = carousels.filterIsInstance<Scorable>()
    return ranker.rank(items, listOf(VerticalStepType.RANK_ALL), ctx)
}
```

## Phase 1: Horizontal Ranking — Same Engine

```kotlin
// ── Domain change (repeat for all 3 horizontal types) ─────────

data class ItemStoreEntity(
    override val id: String,
    override val predictionScore: Double?,
    // ...
) : BaseItemFeature, Scorable {
    override fun withPredictionScore(score: Double) = copy(predictionScore = score)
}


// ── Step: wraps StoreCarouselService sort logic ───────────────

class HorizontalRankAllStep(
    private val storeCarouselService: StoreCarouselService,
) : RankingStep<HorizontalStepType> {
    override val stepType = HorizontalStepType.RANK_ALL

    override suspend fun execute(items: List<Scorable>, ctx: RankingContext): List<Scorable> {
        val sorted = storeCarouselService.sortEntities(ctx, items)
        return sorted.filterIsInstance<Scorable>()
    }
}


// ── Wiring + incision point ───────────────────────────────────

@Bean fun horizontalRanker(svc: StoreCarouselService, metrics: MetricsClient) =
    Ranker<HorizontalStepType>(mapOf(HorizontalStepType.RANK_ALL to HorizontalRankAllStep(svc)), metrics)

suspend fun rankStoresInCarousel(
    stores: List<StoreEntity>,
    ctx: RankingContext,
    ranker: Ranker<HorizontalStepType>,
): List<StoreEntity> {
    val items = stores.filterIsInstance<Scorable>()
    return ranker.rank(items, listOf(HorizontalStepType.RANK_ALL), ctx).filterIsInstance<StoreEntity>()
}
```

**Same `Ranker`, same chain, same handler types.** Only the step enum and step implementation differ. Compiler prevents mixing vertical steps into a horizontal pipeline.

---

## Future Extensions

### Granular steps (Phase 2)

Engine unchanged. Add enum values + step implementations + new pipeline config.

```kotlin
enum class VerticalStepType {
    RANK_ALL, MODEL_SCORING, MULTIPLIER_BOOST, DIVERSITY_RERANK, POSITION_BOOSTING, FIXED_PINNING,
}

class ModelScoringStep(private val scorer: EntityScorer) : RankingStep<VerticalStepType> {
    override val stepType = VerticalStepType.MODEL_SCORING
    override suspend fun execute(items: List<Scorable>, ctx: RankingContext): List<Scorable> {
        val scores = scorer.score(ctx, items)
        return items.map { it.withPredictionScore(scores[it.id] ?: it.predictionScore ?: 0.0) }
    }
}

val granularPipeline = listOf(
    VerticalStepType.MODEL_SCORING, VerticalStepType.MULTIPLIER_BOOST,
    VerticalStepType.DIVERSITY_RERANK, VerticalStepType.POSITION_BOOSTING,
    VerticalStepType.FIXED_PINNING,
)
```

### Conditional execution

```kotlin
class ConditionalHandler(
    private val condition: (RankingContext) -> Boolean,
    private val wrapped: RankingHandler,
) : BaseHandler() {
    override suspend fun handle(items: List<Scorable>, ctx: RankingContext) =
        next(if (condition(ctx)) wrapped.handle(items, ctx) else items, ctx)
}
```

### Shadow mode

```kotlin
class ShadowHandler(
    private val legacy: RankingHandler,
    private val candidate: RankingHandler,
    private val metrics: MetricsClient,
) : BaseHandler() {
    override suspend fun handle(items: List<Scorable>, ctx: RankingContext): List<Scorable> {
        val legacyResult = legacy.handle(items, ctx)
        val candidateResult = candidate.handle(items, ctx)
        metrics.gauge("ranking.shadow.order_match",
            if (legacyResult.map { it.id } == candidateResult.map { it.id }) 1.0 else 0.0)
        return next(legacyResult, ctx)
    }
}
```

### Step params (experiment-driven config)

```kotlin
data class ModelScoringParams(val modelId: String = "sibyl_v3", val calibrationEnabled: Boolean = true)

// Pipeline from experiment resolution:
// [{ type: MODEL_SCORING, params: { modelId: "sibyl_v4_candidate" } }, ...]
```

### DAG (Phase 3)

```
Phase 1:  RANK_ALL
Phase 2:  MODEL_SCORING → MULTIPLIER_BOOST → DIVERSITY_RERANK → POSITION_BOOSTING → FIXED_PINNING
Phase 3:                ┌→ MODEL_A ─┐
          FEATURE_PREP ─┤           ├→ MERGE → DIVERSITY_RERANK → PINNING
                        └→ MODEL_B ─┘
```

Handler interface unchanged. Engine adds topological sort + concurrent execution via coroutines.

---

## Open Questions

- **LiteStoreCollection**: No `predictionScore` field (uses maps). Add field? Exclude from Scorable?
- **StoreEntity `id: Long`**: Scorable expects `String`. Use `toString()` or change Scorable?
- **Chain lifecycle**: Build once at startup or per-request?
