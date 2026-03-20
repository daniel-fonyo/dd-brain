# POC: Generic Ranking Abstraction — End-to-End Example

> Concrete example using `StoreCarousel` showing the full adapt → rank → apply-back flow with generics, typed metadata keys, layer-aware step types, and stub params.

## Design Decisions Captured

| Decision                          | Choice                                                                                             | Why                                                                                                                                                |
| --------------------------------- | -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Generics vs concrete interfaces   | `Rankable<T>` generic                                                                              | Vertical and horizontal are structurally identical — one engine, one step interface, zero duplication                                              |
| Adapters vs direct implementation | Adapter wrapper classes                                                                            | Domain classes are data classes with 50+ params — can't add `var score` or `MutableMap metadata` without breaking `equals`/`hashCode`/`copy`       |
| Metadata map keys                 | `MetadataKey<T>` typed keys                                                                        | String keys are a bug factory — typed keys give compile-time safety and IDE autocomplete                                                           |
| Step type identifier              | Generic `S : Enum<S>` — each layer defines its own enum (`VerticalStepType`, `HorizontalStepType`) | Type system prevents mixing vertical steps into horizontal registry. Each layer's steps are self-documenting.                                      |
| StepParams in Phase 1             | Empty stubs — no data, no logic                                                                    | Existing methods read their own config from DVs/runtime JSON internally. Params become real when we migrate to config-driven experiments (future). |
| Step implementation               | Thin delegation to existing Spring beans                                                           | Steps call `BlendingUtil.blendBundle()`, `EntityScorer.score()`, etc. directly. No new logic.                                                      |

---

## Full Example Code

```kotlin
// ═══════════════════════════════════════════════════════════════
// 1. CORE ABSTRACTIONS — generic, shared between vertical & horizontal
// ═══════════════════════════════════════════════════════════════

// ── Typed metadata keys ─────────────────────────────────────
// Instead of row.metadata["calibrationMultiplier"] (stringly-typed),
// use row.metadata[Keys.CALIBRATION_MULTIPLIER] (compile-time safe).

class MetadataKey<T>(val name: String)

// Type-safe extensions on the metadata map
@Suppress("UNCHECKED_CAST")
operator fun <T> MutableMap<String, Any>.get(key: MetadataKey<T>): T? = this[key.name] as? T
operator fun <T : Any> MutableMap<String, Any>.set(key: MetadataKey<T>, value: T) { this[key.name] = value }

// ── Step type enums ──────────────────────────────────────────
// Each layer defines its own enum. The type system prevents
// mixing vertical steps into a horizontal registry (or vice versa).
// MODEL_SCORING appears in both — it's a valid step in both contexts.

enum class VerticalStepType {
    MODEL_SCORING,
    MULTIPLIER_BOOST,
    DIVERSITY_RERANK,
    POSITION_BOOSTING,
    FIXED_PINNING,
}

enum class HorizontalStepType {
    MODEL_SCORING,
    SCORE_MODIFIER,
    CAMPAIGN_SORT,
    BUSINESS_RULES_SORT,
    ORDER_HISTORY_RERANK,
}

// ── Step params ─────────────────────────────────────────────
// Phase 1: empty stubs. Existing methods read their own config
// from DVs / runtime JSON / hardcoded constants internally.
//
// Future: these become real data classes holding config that
// experiments can override via JSON. That migration happens
// AFTER interfaces are proven in production.

sealed class StepParams

object ModelScoringParams : StepParams()
object MultiplierBoostParams : StepParams()
object DiversityRerankParams : StepParams()
object PositionBoostingParams : StepParams()
object FixedPinningParams : StepParams()

// ── The generic contract ────────────────────────────────────
// Two type params: R = rankable item type, S = step type enum.
// This means Ranker<FeedRow, VerticalStepType> can't accept
// a HorizontalStepType step — the compiler prevents it.

interface Rankable<T : Enum<T>> {
    val id: String
    val type: T
    var score: Double
    val metadata: MutableMap<String, Any>
}

interface RankingStep<R : Rankable<*>, S : Enum<S>> {
    val stepType: S
    val paramsClass: Class<out StepParams>
    suspend fun process(rows: MutableList<R>, context: RankingContext, params: StepParams)
}

class Ranker<R : Rankable<*>, S : Enum<S>>(
    private val stepRegistry: Map<S, RankingStep<R, S>>,
) {
    suspend fun rank(rows: MutableList<R>, pipeline: List<StepConfig<S>>, context: RankingContext): List<R> {
        for (stepConfig in pipeline) {
            val step = stepRegistry[stepConfig.type] ?: continue
            val params = deserialize(stepConfig.rawParams, step.paramsClass)
            step.process(rows, context, params)
        }
        return rows.sortedByDescending { it.score }
    }
}

data class StepConfig<S : Enum<S>>(
    val type: S,                      // layer-specific enum
    val rawParams: String = "{}",     // Phase 1: always empty JSON
)


// ═══════════════════════════════════════════════════════════════
// 2. VERTICAL TYPES
// ═══════════════════════════════════════════════════════════════

enum class RowType {
    STORE_CAROUSEL, ITEM_CAROUSEL, DEAL_CAROUSEL,
    STORE_COLLECTION, COLLECTION_V2, ITEM_COLLECTION,
    MAP_CAROUSEL, REELS_CAROUSEL, STORE_ENTITY,
}

typealias FeedRow = Rankable<RowType>
typealias FeedRowRankingStep = RankingStep<FeedRow, VerticalStepType>


// ═══════════════════════════════════════════════════════════════
// 3. ADAPTER — StoreCarousel → FeedRow
//
//    The ONLY place that knows about StoreCarousel.
//    Everything downstream just sees FeedRow.
// ═══════════════════════════════════════════════════════════════

class StoreCarouselFeedRow(
    val carousel: StoreCarousel,     // accessible to steps that need the original
) : FeedRow {
    override val id: String = carousel.id
    override val type = RowType.STORE_CAROUSEL
    override var score: Double = carousel.predictionScore ?: 0.0
    override val metadata: MutableMap<String, Any> = mutableMapOf()

    // Write final score back → new StoreCarousel via copy()
    fun result(): StoreCarousel = carousel.copy(predictionScore = score)
}


// ═══════════════════════════════════════════════════════════════
// 4. STEPS — thin delegation to EXISTING methods
//
//    Phase 1: steps call the same Spring beans with the same args.
//    The existing methods read their own config internally.
//    We're wrapping them behind RankingStep, not rewriting them.
//
//    What each existing method reads internally:
//      EntityScorer      → P13nRuntimeUtil.getDVTreatmentSibylModelMappingConfig()
//      BlendingUtil      → P13nRuntimeUtil.getVerticalBlendingConfigMap()
//      BoostingBundle    → PersonalizationRuntimeUtil.boostByPositionCarouselIdAllowList()
// ═══════════════════════════════════════════════════════════════

class ModelScoringStep(
    private val entityScorer: EntityScorer,  // existing Spring bean
) : FeedRowRankingStep {
    override val stepType = VerticalStepType.MODEL_SCORING
    override val paramsClass = ModelScoringParams::class.java

    override suspend fun process(rows: MutableList<FeedRow>, context: RankingContext, params: StepParams) {
        // Bridge: FeedRow → ScorableEntity (what entityScorer expects)
        val entities = rows.toScorableEntities()

        // Delegate — entityScorer reads its own model config from
        // P13nRuntimeUtil.getDVTreatmentSibylModelMappingConfig() internally
        val scoreBundle: ScoreBundle = entityScorer.score(context, entities)

        // Bridge back: ScoreBundle → FeedRow scores
        rows.forEach { row ->
            row.score = scoreBundle.scores[row.id] ?: row.score
        }
    }
}

class MultiplierBoostStep(
    private val blendingUtil: BlendingUtil,  // existing Spring bean
) : FeedRowRankingStep {
    override val stepType = VerticalStepType.MULTIPLIER_BOOST
    override val paramsClass = MultiplierBoostParams::class.java

    override suspend fun process(rows: MutableList<FeedRow>, context: RankingContext, params: StepParams) {
        // Bridge: FeedRow → what blendBundle() expects
        val entities = rows.toScorableEntities()
        val scoreBundle = rows.toScoreBundle()

        // Delegate — blendBundle reads its own config from
        // P13nRuntimeUtil.getVerticalBlendingConfigMap() internally
        val result: ScoreBundle = blendingUtil.blendBundle(context, entities, scoreBundle)

        // Bridge back: updated ScoreBundle → FeedRow scores
        rows.forEach { row ->
            row.score = result.scores[row.id] ?: row.score
        }
    }
}

class PositionBoostingStep(
    private val boostingUtil: BoostingBundle,  // existing Spring bean
) : FeedRowRankingStep {
    override val stepType = VerticalStepType.POSITION_BOOSTING
    override val paramsClass = PositionBoostingParams::class.java

    override suspend fun process(rows: MutableList<FeedRow>, context: RankingContext, params: StepParams) {
        // Bridge: FeedRow → BoostingBundle input
        val bundle = rows.toBoostingBundle()

        // Delegate — reads allow-lists from
        // PersonalizationRuntimeUtil.boostByPositionCarouselIdAllowList() internally
        val boosted = bundle.boosted(
            { it.getBoostingHint(context) },
            context.pageType,
            experimentMap = context.experimentMap,
        )

        // Bridge back: boosted scores → FeedRow
        rows.applyBoostingBundle(boosted)
    }
}


// ═══════════════════════════════════════════════════════════════
// 5. SPRING WIRING — one-time setup
// ═══════════════════════════════════════════════════════════════

@Configuration
class UbpConfig {
    @Bean
    fun verticalStepRegistry(
        entityScorer: EntityScorer,
        blendingUtil: BlendingUtil,
        boostingUtil: BoostingBundle,
    ): Map<VerticalStepType, FeedRowRankingStep> = mapOf(
        VerticalStepType.MODEL_SCORING     to ModelScoringStep(entityScorer),
        VerticalStepType.MULTIPLIER_BOOST  to MultiplierBoostStep(blendingUtil),
        VerticalStepType.POSITION_BOOSTING to PositionBoostingStep(boostingUtil),
        // VerticalStepType.DIVERSITY_RERANK  to DiversityRerankStep(...),
        // VerticalStepType.FIXED_PINNING     to FixedPinningStep(...),
    )

    @Bean
    fun verticalRanker(
        verticalStepRegistry: Map<VerticalStepType, FeedRowRankingStep>,
    ): Ranker<FeedRow, VerticalStepType> {
        return Ranker(verticalStepRegistry)
    }
}


// ═══════════════════════════════════════════════════════════════
// 6. THE PIPELINE CONFIG
//    Hardcoded here for illustration. In production, this comes
//    from upstream: hash-based traffic bucketing resolves which
//    experiment config (if any) applies to this request, then
//    config resolution produces the step sequence + params.
//    See: experiment-traffic-industry-research.md, experiment-config-contract.md
//
//    Phase 1: just the step sequence. No params (methods read
//    their own config). Pipeline mirrors what
//    BaseEntityRankerConfiguration.rank() does today.
// ═══════════════════════════════════════════════════════════════

val verticalPipeline = listOf(
    StepConfig(type = VerticalStepType.MODEL_SCORING),        // wraps EntityScorer.score()
    StepConfig(type = VerticalStepType.MULTIPLIER_BOOST),     // wraps BlendingUtil.blendBundle()
    StepConfig(type = VerticalStepType.DIVERSITY_RERANK),     // wraps BlendingUtil.rerankEntitiesWithDiversity()
    StepConfig(type = VerticalStepType.POSITION_BOOSTING),    // wraps BoostingBundle.boosted()
    StepConfig(type = VerticalStepType.FIXED_PINNING),        // wraps pin/flow separation
)


// ═══════════════════════════════════════════════════════════════
// 7. THE INCISION POINT — inside DefaultHomePagePostProcessor
//
//    Read top to bottom: adapt → rank → apply back.
//    This replaces the entire method chain:
//      rankAndDedupeContent → rankAndMergeContent → rankContent
//        → BaseEntityRankerConfiguration.rank()
//          → getEntities/getScoreBundle/getBoostBundle/getRankingBundle
// ═══════════════════════════════════════════════════════════════

suspend fun reOrderGlobalEntitiesV2(
    carousels: List<Any>,       // mixed: StoreCarousel, ItemCarousel, DealCarousel, ...
    context: RankingContext,
    verticalRanker: Ranker<FeedRow, VerticalStepType>,
    pipeline: List<StepConfig<VerticalStepType>>,
): List<Any> {

    // ── ADAPT ───────────────────────────────────────────────
    // 9 domain types → 1 uniform FeedRow.
    // This is the ONLY place with type-checks in the entire pipeline.
    val feedRows: MutableList<FeedRow> = carousels.map { carousel ->
        when (carousel) {
            is StoreCarousel          -> StoreCarouselFeedRow(carousel)
            is ItemCarousel           -> ItemCarouselFeedRow(carousel)
            is DealCarousel           -> DealCarouselFeedRow(carousel)
            is LiteStoreCollection    -> StoreCollectionFeedRow(carousel)
            is CollectionV2           -> CollectionV2FeedRow(carousel)
            is ItemCollection         -> ItemCollectionFeedRow(carousel)
            is MapCarousel            -> MapCarouselFeedRow(carousel)
            is AnnouncementCollection -> ReelsCarouselFeedRow(carousel)
            is StoreEntity            -> StoreEntityFeedRow(carousel)
            else -> throw IllegalArgumentException("Unknown type: ${carousel::class}")
        }
    }.toMutableList()

    // ── RANK ────────────────────────────────────────────────
    // One line. Replaces the entire BaseEntityRankerConfiguration.rank() chain.
    // No type-checks. No scattered utility calls. Just step dispatch.
    val ranked: List<FeedRow> = verticalRanker.rank(feedRows, pipeline, context)

    // ── APPLY BACK ──────────────────────────────────────────
    // FeedRow → original domain objects, in ranked order, with updated scores.
    return ranked.map { row ->
        when (row) {
            is StoreCarouselFeedRow       -> row.result()
            is ItemCarouselFeedRow        -> row.result()
            is DealCarouselFeedRow        -> row.result()
            is StoreCollectionFeedRow     -> row.result()
            is CollectionV2FeedRow        -> row.result()
            is ItemCollectionFeedRow      -> row.result()
            is MapCarouselFeedRow         -> row.result()
            is ReelsCarouselFeedRow       -> row.result()
            is StoreEntityFeedRow         -> row.result()
            else -> throw IllegalArgumentException("Unknown adapter: ${row::class}")
        }
    }
}
```

---

## What Changed from Previous Drafts

| # | Before | After | Why |
|---|---|---|---|
| 1 | `StepParams` with real fields (`predictorRef`, `calibrationEnabled`, etc.) | Empty stub objects (`object ModelScoringParams : StepParams()`) | Existing methods read their own config from DVs/runtime JSON internally. Params with data is a future concern for config-driven experiments. |
| 2 | Steps reimplemented scoring/blending math | Steps delegate to existing Spring beans (`entityScorer.score()`, `blendingUtil.blendBundle()`) | Steps are thin wrappers, not rewrites. Same class, same method, same args. |
| 3 | `StepType` flat enum with comments | Generic `S : Enum<S>` — separate `VerticalStepType` and `HorizontalStepType` enums | Type system prevents mixing steps across layers. Each layer's valid steps are self-documenting. `MODEL_SCORING` appears in both. |
| 4 | `metadata` pre-populated by adapter with calibration/boost values | `metadata` starts empty | Steps delegate to existing methods that read their own config — no need to pre-populate metadata for Phase 1. |
| 5 | `metadata: MutableMap<String, Any>` with string keys | `MetadataKey<T>` typed keys + extensions | String keys → typos, wrong types, no autocomplete. Typed keys catch errors at compile time. Kept for future use. |

## Phase 1 vs Future

| Concern | Phase 1 (this RFC) | Future |
|---|---|---|
| **StepParams** | Empty stubs — `object ModelScoringParams : StepParams()` | Real data classes with config fields. Experiments override via JSON. |
| **Config source** | Each method reads its own DVs/runtime JSON internally | Params injected externally by experiment resolution layer |
| **MetadataKey** | Available but mostly unused — steps delegate to existing methods | Steps read/write typed metadata for inter-step communication |
| **Pipeline config** | Fixed step sequence mirroring `BaseEntityRankerConfiguration.rank()` | Experiment-driven: hash-based bucketing → config resolution → step sequence + params |

## Open Questions

- Should `MetadataKey` instances live in a single `Keys` object, or should each step define its own output keys?
- Should the `when` blocks in adapt/apply-back be replaced with a registry pattern (e.g., `Map<KClass<*>, (Any) -> FeedRow>`) to avoid the exhaustive match?
- Bridge functions (`toScorableEntities()`, `toScoreBundle()`, `applyBoostingBundle()`) are the real implementation work — how complex will these conversions be?
