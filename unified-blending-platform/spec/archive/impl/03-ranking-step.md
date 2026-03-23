# Part 3: Ranking Step Interface and Implementations

## Goal

Define the `FeedRowRankingStep` interface (Strategy pattern) and implement the 2 concrete
vertical ranking steps the engine dispatches to. Steps are stateless — all mutable state is on
the `FeedRow` objects. No step reads DV keys internally — all params flow through typed
`StepParams` injected by the engine after deserializing the step's `rawParams: ObjectNode`.

**Key principle:** We are NOT boiling the ocean. This is step 1. Finer decomposition of steps
is a future iteration once interfaces are proven.

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
  └── BoostAndRankStep      (type = "BOOST_AND_RANK")
```

## Is-a / Has-a Analysis

```
FeedRowRankingStep  IS A strategy algorithm         → interface
ModelScoringStep    IS A FeedRowRankingStep         → implements
ModelScoringStep    HAS A EntityScorer              → DI (wraps getScoreBundle())
BoostAndRankStep    IS A FeedRowRankingStep         → implements
BoostAndRankStep    HAS A BoostingUtil              → DI (wraps getBoostBundle() + getRankingBundle())
```

Steps never extend domain classes. They receive `FeedRow` (the abstraction), never
`StoreCarousel`. DI for collaborators keeps steps testable in isolation.

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

Wraps `getScoreBundle()` — Sibyl gRPC + `BlendingUtil.blendBundle()` including diversity
rerank, all as one atomic call. This is intentionally coarse: the Sibyl call and blending
are tightly coupled in the existing code and splitting them prematurely adds risk without
proven value.

```kotlin
// libraries/common/.../ubp/vertical/steps/ModelScoringStep.kt
class ModelScoringStep(
    private val entityScorer: EntityScorer,
) : FeedRowRankingStep {
    override val stepType   = "MODEL_SCORING"
    override val paramsClass = Params::class.java

    data class Params(
        @JsonProperty("predictor_ref")
        val predictorRef: String,               // resolved to PredictorConfig by engine
        @JsonProperty("predictor_name")
        val predictorName: String,
        @JsonProperty("model_name")
        val modelName: String,
    ) : StepParams

    override suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: StepParams,
    ) {
        val p = params as Params   // safe: engine used Params::class.java

        // Wraps getScoreBundle() — Sibyl gRPC + BlendingUtil.blendBundle() + diversity rerank
        val scoreBundle = entityScorer.getScoreBundle(
            ids           = rows.map { it.id },
            predictorName = p.predictorName,
            modelName     = p.modelName,
            context       = context,
        )

        rows.forEach { row -> row.score = scoreBundle.scoreFor(row.id) ?: 0.0 }
    }
}
```

**Params contract:**
```
predictor_ref                String            required  e.g. "p_act"
predictor_name               String            required  e.g. "feed_ranking", "feed_ranking_fw"
model_name                   String            required  e.g. "store_ranking_v1_1"
```

---

## BoostAndRankStep (type = "BOOST_AND_RANK")

Wraps `getBoostBundle()` + `getRankingBundle()` + `getRankableContent()` — score assignment
to domain objects, position boosting, deal multiplier, pin vs flow sort order, reassembly.
All as one atomic flow. This is intentionally coarse: these operations are tightly coupled
in the existing code.

Does NOT replace post-ranking business fixups:
- `NonRankableHomepageOrderingUtil.updateSortOrderOfNVCarousel()` (post-checkout pin, context-driven)
- `updateSortOrderOfTasteOfDashPass()` (PAD=3, owned by its own DV)
- `updateSortOrderOfMemberPricing()` (runtime feature flag)

```kotlin
// libraries/common/.../ubp/vertical/steps/BoostAndRankStep.kt
class BoostAndRankStep(
    private val boostingUtil: BoostingUtil,
) : FeedRowRankingStep {
    override val stepType    = "BOOST_AND_RANK"
    override val paramsClass = Params::class.java

    data class Params(
        @JsonProperty("boost_by_position_enabled")
        val boostByPositionEnabled: Boolean = false,

        @JsonProperty("deal_carousel_score_multiplier")
        val dealCarouselScoreMultiplier: Double = 1.0,

        @JsonProperty("boost_by_position_allow_list")
        val boostByPositionAllowList: List<String> = emptyList(),
    ) : StepParams

    override suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: StepParams,
    ) {
        val p = params as Params

        // Wraps getBoostBundle() + getRankingBundle() + getRankableContent()
        // Score assignment, position boosting, deal multiplier, pin vs flow sort, reassembly
        val boostBundle = boostingUtil.getBoostBundle(rows, p)
        val rankingBundle = boostingUtil.getRankingBundle(rows, p)
        val rankableContent = boostingUtil.getRankableContent(rows, boostBundle, rankingBundle)

        rows.clear()
        rows.addAll(rankableContent)
    }
}
```

**Params contract:**
```
boost_by_position_enabled         Boolean        default false
deal_carousel_score_multiplier    Double         default 1.0
boost_by_position_allow_list      List<String>   default []
```

---

## Files to Create

| File | Purpose |
|---|---|
| `libraries/common/.../ubp/vertical/FeedRowRankingStep.kt` | Interface |
| `libraries/common/.../ubp/vertical/steps/ModelScoringStep.kt` | Step + Params |
| `libraries/common/.../ubp/vertical/steps/BoostAndRankStep.kt` | Step + Params |

---

## Testing

```kotlin
@Test
fun `ModelScoringStep applies Sibyl score to rows`() {
    val scorer = mockk<EntityScorer>()
    coEvery { scorer.getScoreBundle(any(), "feed_ranking", "v1", any()) } returns fakeScoreBundle("r1" to 0.8)

    val step = ModelScoringStep(scorer)
    val rows = mutableListOf(fakeRow("r1", initialScore = 0.0))
    val params = ModelScoringStep.Params(
        predictorRef = "p_act", predictorName = "feed_ranking", modelName = "v1"
    )
    runBlocking { step.process(rows, fakeContext(), params) }
    assertEquals(0.8, rows[0].score, 0.001)
}

@Test
fun `BoostAndRankStep applies boost and rank logic`() {
    val boostingUtil = mockk<BoostingUtil>()
    // Mock the three wrapped calls
    coEvery { boostingUtil.getBoostBundle(any(), any()) } returns fakeBoostBundle()
    coEvery { boostingUtil.getRankingBundle(any(), any()) } returns fakeRankingBundle()
    coEvery { boostingUtil.getRankableContent(any(), any(), any()) } returns listOf(
        fakeRow("r2", score = 0.9), fakeRow("r1", score = 0.5)
    )

    val step = BoostAndRankStep(boostingUtil)
    val rows = mutableListOf(fakeRow("r1", score = 0.5), fakeRow("r2", score = 0.9))
    val params = BoostAndRankStep.Params(
        boostByPositionEnabled = false,
        dealCarouselScoreMultiplier = 1.0,
    )
    runBlocking { step.process(rows, fakeContext(), params) }
    assertEquals(listOf("r2", "r1"), rows.map { it.id })
}

@Test
fun `BoostAndRankStep is no-op with default params`() {
    val boostingUtil = mockk<BoostingUtil>()
    coEvery { boostingUtil.getBoostBundle(any(), any()) } returns fakeBoostBundle()
    coEvery { boostingUtil.getRankingBundle(any(), any()) } returns fakeRankingBundle()
    coEvery { boostingUtil.getRankableContent(any(), any(), any()) } returns listOf(fakeRow("s1"), fakeRow("s2"))

    val step = BoostAndRankStep(boostingUtil)
    val rows = mutableListOf(fakeRow("s1"), fakeRow("s2"))
    runBlocking { step.process(rows, fakeContext(), BoostAndRankStep.Params()) }
    assertEquals(listOf("s1", "s2"), rows.map { it.id })
}
```

---

## Value Function Mapping

```
Full EV  = pImp(k)  ×   pAct(c)          ×   vAct(c)
Phase 1  =   1.0    ×  MODEL_SCORING     ×  BOOST_AND_RANK
```

- **`MODEL_SCORING`** → `pAct(c)` via Sibyl + blending (calibration, intent, boost weights, diversity rerank — all atomic)
- **`BOOST_AND_RANK`** → `vAct(c)` proxy via score assignment, position boosting, deal multiplier, pin vs flow sort order
- **`pImp(k)`** = 1.0 in Phase 1 — steps have no knowledge of final position. Phase 3.
- **Step order is load-bearing:** `MODEL_SCORING` must precede `BOOST_AND_RANK`

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

- Both step classes created and unit tested (`ModelScoringStep`, `BoostAndRankStep`)
- Each step has a typed `Params` data class with `@JsonProperty` annotations — no `Map<String, Any>`
- No step reads a DV key internally
- No step casts `params["key"] as Type` — all param access through typed fields
- `ModelScoringStep` wraps `getScoreBundle()` as one atomic call (Sibyl + blending + diversity)
- `BoostAndRankStep` wraps `getBoostBundle()` + `getRankingBundle()` + `getRankableContent()` as one atomic flow
