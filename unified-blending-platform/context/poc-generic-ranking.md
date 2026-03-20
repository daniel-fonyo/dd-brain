# POC: Generic Ranking Abstraction — End-to-End Example

> Concrete example using `StoreCarousel` showing the full adapt → rank → apply-back flow with generics, typed metadata keys, and enum step types.

## Design Decisions Captured

| Decision | Choice | Why |
|---|---|---|
| Generics vs concrete interfaces | `Rankable<T>` generic | Vertical and horizontal are structurally identical — one engine, one step interface, zero duplication |
| Adapters vs direct implementation | Adapter wrapper classes | Domain classes are data classes with 50+ params — can't add `var score` or `MutableMap metadata` without breaking `equals`/`hashCode`/`copy` |
| Metadata map keys | `MetadataKey<T>` typed keys | String keys are a bug factory — typed keys give compile-time safety and IDE autocomplete |
| Step type identifier | `StepType` enum | String identifiers lead to typos and missing references — enum catches errors at compile time |

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

object Keys {
    // Populated by adapters (from domain object fields)
    val SCORE_MULTIPLIER        = MetadataKey<Double>("scoreMultiplier")
    val CALIBRATION_MULTIPLIER  = MetadataKey<Double>("calibrationMultiplier")
    val VERTICAL_BOOST_WEIGHT   = MetadataKey<Double>("verticalBoostWeight")
    val INTENT_MULTIPLIER       = MetadataKey<Double>("verticalIntentMultiplier")

    // Written by steps (for downstream steps or tracing)
    val APPLIED_CALIBRATION     = MetadataKey<Double>("appliedCalibration")
    val APPLIED_INTENT          = MetadataKey<Double>("appliedIntent")
    val APPLIED_BOOST_WEIGHT    = MetadataKey<Double>("appliedBoostWeight")
}

// Type-safe extensions on the metadata map
@Suppress("UNCHECKED_CAST")
operator fun <T> MutableMap<String, Any>.get(key: MetadataKey<T>): T? = this[key.name] as? T
operator fun <T : Any> MutableMap<String, Any>.set(key: MetadataKey<T>, value: T) { this[key.name] = value }

// ── Step type enum ──────────────────────────────────────────
// Single enum for both layers. MODEL_SCORING is shared.

enum class StepType {
    // Vertical
    MODEL_SCORING,
    MULTIPLIER_BOOST,
    DIVERSITY_RERANK,
    POSITION_BOOSTING,
    FIXED_PINNING,
    // Horizontal
    SCORE_MODIFIER,
    CAMPAIGN_SORT,
    BUSINESS_RULES_SORT,
    ORDER_HISTORY_RERANK,
}

// ── The generic contract ────────────────────────────────────

interface Rankable<T : Enum<T>> {
    val id: String
    val type: T
    var score: Double
    val metadata: MutableMap<String, Any>
}

interface RankingStep<R : Rankable<*>> {
    val stepType: StepType
    val paramsClass: Class<out StepParams>
    suspend fun process(rows: MutableList<R>, context: RankingContext, params: StepParams)
}

class Ranker<R : Rankable<*>>(
    private val stepRegistry: Map<StepType, RankingStep<R>>,
) {
    suspend fun rank(rows: MutableList<R>, pipeline: List<StepConfig>, context: RankingContext): List<R> {
        for (stepConfig in pipeline) {
            val step = stepRegistry[stepConfig.type] ?: continue
            val params = deserialize(stepConfig.rawParams, step.paramsClass)
            step.process(rows, context, params)
        }
        return rows.sortedByDescending { it.score }
    }
}

data class StepConfig(
    val type: StepType,       // enum, not string
    val rawParams: String,    // JSON — deserialized into step's paramsClass
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
typealias FeedRowRankingStep = RankingStep<FeedRow>


// ═══════════════════════════════════════════════════════════════
// 3. ADAPTER — StoreCarousel → FeedRow
//
//    The ONLY place that knows about StoreCarousel.
//    Everything downstream just sees FeedRow.
// ═══════════════════════════════════════════════════════════════

class StoreCarouselFeedRow(
    private val carousel: StoreCarousel,
) : FeedRow {
    override val id: String = carousel.id
    override val type = RowType.STORE_CAROUSEL
    override var score: Double = carousel.predictionScore ?: 0.0
    override val metadata: MutableMap<String, Any> = mutableMapOf<String, Any>().apply {
        // Type-safe — compiler enforces Double
        set(Keys.SCORE_MULTIPLIER, carousel.scoreMultiplier)
        set(Keys.CALIBRATION_MULTIPLIER, carousel.calibrationMultiplier ?: 1.0)
        set(Keys.VERTICAL_BOOST_WEIGHT, carousel.verticalBoostWeight ?: 1.0)
        set(Keys.INTENT_MULTIPLIER, carousel.verticalIntentMultiplier ?: 1.0)
    }

    // Write final score back → new StoreCarousel via copy()
    fun result(): StoreCarousel = carousel.copy(predictionScore = score)
}


// ═══════════════════════════════════════════════════════════════
// 4. STEP PARAMS — mirrors existing feed-service config classes
// ═══════════════════════════════════════════════════════════════

sealed class StepParams

data class ModelScoringParams(
    val predictorRef: String,            // from VerticalBlendingConfig
    val predictorName: String,           // from VerticalBlendingConfig
    val modelName: String,               // from VerticalBlendingConfig
) : StepParams()

data class MultiplierBoostParams(
    val calibrationEnabled: Boolean,     // from CalibrationConfig
    val intentScoringEnabled: Boolean,   // from IntentScoringConfig
    val verticalBoostWeights: Map<RowType, Double>,  // enum keys, not strings
) : StepParams()


// ═══════════════════════════════════════════════════════════════
// 5. STEPS — thin wrappers around EXISTING methods
//
//    Steps don't rewrite logic. They call the same class,
//    same method, same args. They just read/write through
//    FeedRow instead of ScorableEntity.
// ═══════════════════════════════════════════════════════════════

class ModelScoringStep(
    private val sibylRegressor: SibylRegressor,  // existing Spring bean
) : FeedRowRankingStep {
    override val stepType = StepType.MODEL_SCORING
    override val paramsClass = ModelScoringParams::class.java

    override suspend fun process(rows: MutableList<FeedRow>, context: RankingContext, params: StepParams) {
        val p = params as ModelScoringParams

        // Same RegressionContext that entityScorer.score() builds today
        val regressionContext = RegressionContext(
            predictorName = p.predictorName,
            modelOverrideId = p.modelName,
        )

        // Same gRPC call: SibylRegressor.computeRegressionScores()
        val featureSets = rows.map { it.toFeatureSet() }
        val scores: Map<String, RegressionValue> = sibylRegressor.computeRegressionScores(
            regressionContext,
            featureSets,
        )

        // Write scores onto FeedRow
        rows.forEach { row ->
            row.score = scores[row.id]?.score ?: row.score
        }
    }
}

class MultiplierBoostStep : FeedRowRankingStep {
    override val stepType = StepType.MULTIPLIER_BOOST
    override val paramsClass = MultiplierBoostParams::class.java

    override suspend fun process(rows: MutableList<FeedRow>, context: RankingContext, params: StepParams) {
        val p = params as MultiplierBoostParams

        rows.forEach { row ->
            // Same math as BlendingUtil.scoreEntity() — reads from typed metadata
            val calibration = if (p.calibrationEnabled) {
                row.metadata[Keys.CALIBRATION_MULTIPLIER] ?: 1.0  // compile-time type-safe
            } else 1.0

            val intent = if (p.intentScoringEnabled) {
                row.metadata[Keys.INTENT_MULTIPLIER] ?: 1.0
            } else 1.0

            val boostWeight = p.verticalBoostWeights[row.type] ?: 1.0  // enum key, not string

            row.score *= calibration * intent * boostWeight

            // Stash factors for downstream steps / tracing — typed, not stringly
            row.metadata[Keys.APPLIED_CALIBRATION] = calibration
            row.metadata[Keys.APPLIED_INTENT] = intent
            row.metadata[Keys.APPLIED_BOOST_WEIGHT] = boostWeight
        }
    }
}


// ═══════════════════════════════════════════════════════════════
// 6. SPRING WIRING — one-time setup
// ═══════════════════════════════════════════════════════════════

@Configuration
class UbpConfig {
    @Bean
    fun verticalStepRegistry(sibylRegressor: SibylRegressor): Map<StepType, FeedRowRankingStep> = mapOf(
        StepType.MODEL_SCORING    to ModelScoringStep(sibylRegressor),
        StepType.MULTIPLIER_BOOST to MultiplierBoostStep(),
        // StepType.DIVERSITY_RERANK    to DiversityRerankStep(...),
        // StepType.POSITION_BOOSTING   to PositionBoostingStep(),
        // StepType.FIXED_PINNING       to FixedPinningStep(),
    )

    @Bean
    fun verticalRanker(verticalStepRegistry: Map<StepType, FeedRowRankingStep>): Ranker<FeedRow> {
        return Ranker(verticalStepRegistry)
    }
}


// ═══════════════════════════════════════════════════════════════
// 7. THE PIPELINE CONFIG
//    Hardcoded here for illustration. In production, this comes
//    from upstream: hash-based traffic bucketing resolves which
//    experiment config (if any) applies to this request, then
//    config resolution produces the step sequence + params.
//    See: experiment-traffic-industry-research.md, experiment-config-contract.md
// ═══════════════════════════════════════════════════════════════

val verticalPipeline = listOf(
    StepConfig(
        type = StepType.MODEL_SCORING,
        rawParams = """{"predictorRef":"hp_v2", "predictorName":"hp_scorer", "modelName":"hp_v2_model"}""",
    ),
    StepConfig(
        type = StepType.MULTIPLIER_BOOST,
        rawParams = """{"calibrationEnabled":true, "intentScoringEnabled":true, "verticalBoostWeights":{"STORE_CAROUSEL":1.2, "DEAL_CAROUSEL":1.5}}""",
    ),
)


// ═══════════════════════════════════════════════════════════════
// 8. THE INCISION POINT — inside DefaultHomePagePostProcessor
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
    verticalRanker: Ranker<FeedRow>,
    pipeline: List<StepConfig>,
): List<Any> {

    // ── ADAPT ───────────────────────────────────────────────
    // 9 domain types → 1 uniform FeedRow.
    // This is the ONLY place with type-checks in the entire pipeline.
    val feedRows: MutableList<FeedRow> = carousels.map { carousel ->
        when (carousel) {
            is StoreCarousel       -> StoreCarouselFeedRow(carousel)
            is ItemCarousel        -> ItemCarouselFeedRow(carousel)
            is DealCarousel        -> DealCarouselFeedRow(carousel)
            is LiteStoreCollection -> StoreCollectionFeedRow(carousel)
            is CollectionV2        -> CollectionV2FeedRow(carousel)
            is ItemCollection      -> ItemCollectionFeedRow(carousel)
            is MapCarousel         -> MapCarouselFeedRow(carousel)
            is AnnouncementCollection -> ReelsCarouselFeedRow(carousel)
            is StoreEntity         -> StoreEntityFeedRow(carousel)
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

## What Changed from First Draft

| Before | After | Why |
|---|---|---|
| `metadata: MutableMap<String, Any>` with string keys | `MetadataKey<T>` typed keys + extensions | String keys → typos, wrong types, no autocomplete. Typed keys catch errors at compile time. |
| `stepType: String` | `stepType: StepType` (enum) | Strings are invisible to the compiler. Enum gives exhaustive matching, refactor safety, IDE support. |
| `verticalBoostWeights: Map<String, Double>` | `Map<RowType, Double>` | Same principle — enum keys instead of string keys. |

## Open Questions

- Should `MetadataKey` instances live in a single `Keys` object, or should each step define its own output keys?
- Should the `when` blocks in adapt/apply-back be replaced with a registry pattern (e.g., `Map<KClass<*>, (Any) -> FeedRow>`) to avoid the exhaustive match?
