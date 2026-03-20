# POC: Scorable Interface — No Adapters, External ScoreBoard

> Domain objects implement `Scorable` directly (one-line change per class). No wrapper/adapter classes. Mutable scores tracked in an external `ScoreBoard`. Phase 1 uses one coarse `RANK_ALL` step wrapping the entire legacy pipeline.

## Design Decisions Captured

| Decision | Choice | Why |
| --- | --- | --- |
| Adapters vs direct interface | Domain objects implement `Scorable` directly | Eliminates 9 adapter wrapper classes. One-line change per data class — add `: Scorable` and `override` existing fields. |
| Mutable score tracking | External `ScoreBoard` in the engine | Data classes stay immutable. No `var score` breaking `equals`/`hashCode`/`copy`. Engine owns mutation. |
| Apply-back mechanism | `withPredictionScore(score): Scorable` on each type | Each data class implements via `copy(predictionScore = score)`. One-liner, co-located. No `when` block needed at apply-back. |
| Generics | `Ranker<S : Enum<S>>`, `RankingStep<S : Enum<S>>` | One engine, one step interface. Type system prevents mixing vertical steps into horizontal registry. |
| Typed metadata keys | `MetadataKey<T>` | Compile-time safety, IDE autocomplete. Unused in Phase 1 but ready for inter-step communication. |
| StepParams in Phase 1 | Empty stubs — no data, no logic | Existing methods read their own config from DVs/runtime JSON internally. Real params are a future concern. |
| Phase 1 granularity | One coarse `RANK_ALL` step | Wraps `BaseEntityRankerConfiguration.rank()` as-is. No bridge conversions. Granular steps come in Phase 2. |

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

    // Each data class implements via copy(predictionScore = score).
    // Eliminates the apply-back `when` block entirely.
    fun withPredictionScore(score: Double): Scorable
}


// ── ScoreBoard — external mutable score tracking ─────────────
// Domain objects stay immutable (val predictionScore).
// The engine tracks mutable scores here instead.
//
// Phase 1 (RANK_ALL): legacy ranker produces final scores,
//   ScoreBoard captures them for apply-back.
// Phase 2 (granular steps): each step reads/writes ScoreBoard,
//   items are never mutated mid-pipeline.

class ScoreBoard(items: List<Scorable>) {
    private val scores = HashMap<String, Double>(items.size)

    init {
        items.forEach { scores[it.id] = it.predictionScore ?: 0.0 }
    }

    operator fun get(id: String): Double = scores[id] ?: 0.0
    operator fun set(id: String, value: Double) { scores[id] = value }
}


// ── Typed metadata keys ──────────────────────────────────────
// Phase 1: unused — steps delegate to existing methods.
// Phase 2: steps communicate via typed metadata (e.g., calibration
// multiplier from scoring step → boost step).

class MetadataKey<T>(val name: String)

@Suppress("UNCHECKED_CAST")
operator fun <T> MutableMap<String, Any>.get(key: MetadataKey<T>): T? = this[key.name] as? T
operator fun <T : Any> MutableMap<String, Any>.set(key: MetadataKey<T>, value: T) { this[key.name] = value }


// ── Step type enums ──────────────────────────────────────────
// Each layer defines its own enum. The type system prevents
// mixing vertical steps into a horizontal registry.

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
// Phase 1: empty stubs. Existing methods read their own config
// from DVs / runtime JSON / hardcoded constants internally.
// Future: real data classes holding config that experiments
// can override via JSON.

sealed class StepParams
object RankAllParams : StepParams()


// ── Step + Ranker contracts ──────────────────────────────────
// Generic on S (step type enum). Ranker<VerticalStepType> can't
// accept a HorizontalStepType step — compiler prevents it.

interface RankingStep<S : Enum<S>> {
    val stepType: S
    val paramsClass: Class<out StepParams>
    suspend fun execute(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
        params: StepParams,
    ): List<Scorable>   // return items in (potentially new) order
}

data class StepConfig<S : Enum<S>>(
    val type: S,
    val rawParams: String = "{}",   // Phase 1: always empty JSON
)

class Ranker<S : Enum<S>>(
    private val stepRegistry: Map<S, RankingStep<S>>,
) {
    suspend fun rank(
        items: List<Scorable>,
        pipeline: List<StepConfig<S>>,
        context: RankingContext,
    ): RankedResult {
        val scoreBoard = ScoreBoard(items)
        var current = items

        for (stepConfig in pipeline) {
            val step = stepRegistry[stepConfig.type] ?: continue
            val params = deserialize(stepConfig.rawParams, step.paramsClass)
            current = step.execute(current, scoreBoard, context, params)
        }

        return RankedResult(current, scoreBoard)
    }
}

data class RankedResult(
    val items: List<Scorable>,
    val scoreBoard: ScoreBoard,
)


// ═══════════════════════════════════════════════════════════════
// 2. DOMAIN CHANGES — one-line per type
//
//    Each data class already has `id` and `predictionScore`.
//    Changes: add `: Scorable`, add `override`, add withPredictionScore.
//    That's it. No new classes. No wrappers.
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

// Same pattern for all other types. Example:
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

// LiteStoreCollection — the outlier. Uses maps instead of single
// predictionScore. Add a predictionScore field defaulting to null,
// or compute from storeScoreMap. Exact approach TBD.
data class LiteStoreCollection(
    override val id: String,
    override val predictionScore: Double? = null,  // may need to add this field
    val storeScoreMap: Map<String, Double> = emptyMap(),
    // ...
) : Scorable {
    override fun withPredictionScore(score: Double) = copy(predictionScore = score)
}


// ═══════════════════════════════════════════════════════════════
// 3. PHASE 1 STEP — RANK_ALL
//
//    Wraps the ENTIRE existing pipeline as one opaque step.
//    No bridge conversions. No new scoring logic.
//    items go in as their original types → legacy ranker handles
//    them exactly as it does today → copies come back with
//    updated scores and correct ordering.
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
        // Legacy ranker already knows how to handle them — same types,
        // same method, same args. Nothing changes inside.
        val ranked: List<Any> = legacyRanker.rank(context, items)

        // Legacy ranker returns new copies (data class copy()) with
        // updated predictionScore. Capture those scores in ScoreBoard.
        val result = ranked.filterIsInstance<Scorable>()
        result.forEach { scoreBoard[it.id] = it.predictionScore ?: 0.0 }

        return result   // already in correct order with correct scores
    }
}


// ═══════════════════════════════════════════════════════════════
// 4. SPRING WIRING — one-time setup
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
    ): Ranker<VerticalStepType> = Ranker(registry)
}


// ═══════════════════════════════════════════════════════════════
// 5. PIPELINE CONFIG
//    Hardcoded for illustration. In production, this comes from
//    upstream: hash-based traffic bucketing resolves which
//    experiment config applies, then config resolution produces
//    the step sequence + params.
//    See: experiment-traffic-industry-research.md
//
//    Phase 1: single RANK_ALL step. Mirrors exactly what
//    BaseEntityRankerConfiguration.rank() does today.
// ═══════════════════════════════════════════════════════════════

val verticalPipeline = listOf(
    StepConfig(type = VerticalStepType.RANK_ALL),
)


// ═══════════════════════════════════════════════════════════════
// 6. THE INCISION POINT — inside DefaultHomePagePostProcessor
//
//    Compare to the adapter POC:
//    - No adapt `when` block (items go in as-is)
//    - No apply-back `when` block (withPredictionScore is polymorphic)
//    - No adapter classes anywhere
//
//    This replaces the entire method chain:
//      rankAndDedupeContent → rankAndMergeContent → rankContent
//        → BaseEntityRankerConfiguration.rank()
// ═══════════════════════════════════════════════════════════════

suspend fun reOrderGlobalEntitiesV2(
    carousels: List<Any>,       // mixed: StoreCarousel, ItemCarousel, DealCarousel, ...
    context: RankingContext,
    verticalRanker: Ranker<VerticalStepType>,
    pipeline: List<StepConfig<VerticalStepType>>,
): List<Any> {

    // ── No adapt step needed ──────────────────────────────────
    // Domain objects already implement Scorable.
    // No wrapper classes, no `when` block, no conversion.
    val items: List<Scorable> = carousels.filterIsInstance<Scorable>()

    // ── RANK ──────────────────────────────────────────────────
    // One line. Same as adapter POC but without adapters.
    val (ranked, scores) = verticalRanker.rank(items, pipeline, context)

    // ── APPLY BACK ────────────────────────────────────────────
    // Phase 1 (RANK_ALL): legacy ranker already produced copies
    // with correct scores. Just return in ranked order.
    return ranked

    // Phase 2 (granular steps): apply scoreBoard back to items.
    // withPredictionScore() is polymorphic — no `when` needed.
    // return ranked.map { it.withPredictionScore(scores[it.id]) }
}
```

---

## Phase 2 Sketch — Granular Steps

When we break `RANK_ALL` into individual steps, the engine stays identical.
Only the step registry and pipeline config change.

```kotlin
// Step types expand (no breaking changes — just add enum values)
enum class VerticalStepType {
    RANK_ALL,
    MODEL_SCORING,
    MULTIPLIER_BOOST,
    DIVERSITY_RERANK,
    POSITION_BOOSTING,
    FIXED_PINNING,
}

// Each step delegates to existing Spring beans.
// The "bridge" between Scorable and legacy API types is simpler
// than the adapter approach because items ARE their original types
// at runtime — just cast when the legacy API needs concrete types.

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
        // Items are still StoreCarousel, ItemCarousel, etc. at runtime.
        // EntityScorer works with their existing types/interfaces.
        val scoreBundle = entityScorer.score(context, items)

        // Write scores to ScoreBoard (not to the items)
        scoreBundle.forEach { (id, score) -> scoreBoard[id] = score }

        return items   // same order — scoring doesn't reorder
    }
}

class DiversityRerankStep(
    private val blendingUtil: BlendingUtil,
) : RankingStep<VerticalStepType> {
    override val stepType = VerticalStepType.DIVERSITY_RERANK
    override val paramsClass = DiversityRerankParams::class.java

    override suspend fun execute(
        items: List<Scorable>,
        scoreBoard: ScoreBoard,
        context: RankingContext,
        params: StepParams,
    ): List<Scorable> {
        // This step reorders items — return in new order
        val reordered = blendingUtil.rerankWithDiversity(context, items, scoreBoard)
        return reordered
    }
}

// Phase 2 pipeline — same Ranker, different config
val verticalPipelineV2 = listOf(
    StepConfig(type = VerticalStepType.MODEL_SCORING),
    StepConfig(type = VerticalStepType.MULTIPLIER_BOOST),
    StepConfig(type = VerticalStepType.DIVERSITY_RERANK),
    StepConfig(type = VerticalStepType.POSITION_BOOSTING),
    StepConfig(type = VerticalStepType.FIXED_PINNING),
)

// Phase 2 incision point — only apply-back changes
val (ranked, scores) = verticalRanker.rank(items, verticalPipelineV2, context)
return ranked.map { it.withPredictionScore(scores[it.id]) }
```

---

## What Changed from Adapter POC

| # | Adapter POC | Scorable POC | Why |
|---|---|---|---|
| 1 | 9 adapter wrapper classes (`StoreCarouselFeedRow`, etc.) | 0 new classes — domain objects implement `Scorable` directly | One-line change per data class. No wrappers, no `result()`, no `when` blocks for adapt. |
| 2 | `var score: Double` on adapter (mutable) | `ScoreBoard` in engine (external mutation) | Data classes stay fully immutable. `equals`/`hashCode`/`copy` untouched. |
| 3 | `fun result(): StoreCarousel = carousel.copy(predictionScore = score)` | `fun withPredictionScore(score: Double) = copy(predictionScore = score)` | Same semantics, but defined on domain class (co-located). Polymorphic — no `when` needed. |
| 4 | `when (carousel) { is StoreCarousel -> StoreCarouselFeedRow(carousel) ... }` adapt block | `carousels.filterIsInstance<Scorable>()` | No type-switching. Items go in as-is. |
| 5 | `when (row) { is StoreCarouselFeedRow -> row.result() ... }` apply-back block | `ranked.map { it.withPredictionScore(scores[it.id]) }` | Polymorphic dispatch. One line. |
| 6 | 5 granular steps with hand-waved bridge functions (`toScorableEntities()`) | 1 coarse `RANK_ALL` step for Phase 1, granular for Phase 2 | Phase 1 avoids bridge complexity entirely. Bridge is simpler in Phase 2 because items ARE original types. |
| 7 | `Rankable<T : Enum<T>>` with `var score`, `metadata`, `type` | `Scorable` with just `id`, `predictionScore`, `withPredictionScore` | Minimal interface. Domain classes don't need `type` enum or `metadata` map. |

## Phase 1 vs Future

| Concern | Phase 1 (this RFC) | Future |
|---|---|---|
| **Step granularity** | One `RANK_ALL` step wrapping entire legacy pipeline | 5 granular vertical steps, 5 horizontal steps |
| **ScoreBoard** | Captures legacy output scores for bookkeeping | Primary score-tracking mechanism — steps read/write it |
| **StepParams** | Empty stubs — `object RankAllParams : StepParams()` | Real data classes with config fields. Experiments override via JSON. |
| **MetadataKey** | Available but unused | Steps communicate via typed metadata (e.g., calibration multiplier) |
| **Pipeline config** | Single `RANK_ALL` step mirroring today's code | Experiment-driven: hash-based bucketing → config resolution → step sequence + params |
| **Apply-back** | Not needed — `RANK_ALL` produces copies with correct scores | `ranked.map { it.withPredictionScore(scores[it.id]) }` |
| **Bridge to legacy APIs** | None — legacy ranker handles original types directly | Steps cast `Scorable` to concrete types where legacy APIs need them |

## Open Questions

- **LiteStoreCollection**: Uses `storeScoreMap` instead of a single `predictionScore`. Add a `predictionScore` field? Compute from map? Exclude from Scorable?
- **withPredictionScore return type**: Returns `Scorable` (not the concrete type). Downstream code that needs `StoreCarousel` must cast. Is this acceptable? (It's the same situation as today — existing code returns `List<Any>` from ranking.)
- **Phase 2 bridge complexity**: When breaking into granular steps, how hard is the cast from `Scorable` to the types each legacy method expects? Easier than the adapter bridge, but still work.
