# Part 3: Ranking Step Interface and Implementations

## Goal

Define the `FeedRowRankingStep` interface (Strategy pattern) and implement the 4 concrete ranking steps the engine dispatches to. Steps are stateless — all mutable state is on the `FeedRow` objects. No step reads DV keys internally — all params flow through the `params: Map<String, Any>` argument.

## Design Pattern: Strategy (Behavioral)

> "Define a family of algorithms, put each of them into a separate class, and make their objects interchangeable." — refactoring.guru

The engine doesn't know which algorithm it's calling — it only knows the `FeedRowRankingStep` interface. Swapping steps (changing the `type` key in config) requires zero engine changes.

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
DiversityRerankStep IS A FeedRowRankingStep         → implements
FixedPinningStep    IS A FeedRowRankingStep         → implements (no collaborators)
MultiplierBoostStep IS A FeedRowRankingStep         → implements (no collaborators)
```

Steps never extend domain classes. They receive `FeedRow` (the abstraction), never `StoreCarousel`. DI for collaborators (`EntityScorer`) keeps the step testable in isolation — no real service needed in tests.

## Interface Definition

```kotlin
// libraries/common/.../ubp/vertical/FeedRowRankingStep.kt
interface FeedRowRankingStep {
    val stepType: String   // matches "type" in config steps array

    suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: Map<String, Any>,
    )
}
```

`suspend`: steps may call async collaborators — model scoring is async. Non-async steps implement this trivially with no cost.

`params: Map<String, Any>`: steps must NOT read DV keys. All tunable behavior comes through params injected by the engine from the experiment config. This is the critical abstraction boundary — experiments are config changes, not code changes.

## ModelScoringStep (type = "MODEL_SCORING")

```kotlin
// libraries/common/.../ubp/vertical/steps/ModelScoringStep.kt
class ModelScoringStep(
    private val entityScorer: EntityScorer,
) : FeedRowRankingStep {
    override val stepType = "MODEL_SCORING"

    override suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: Map<String, Any>,
    ) {
        val predictorName = params["predictor_name"] as String
        val modelName = params["model_name"] as String
        val calibrationMultiplier = params["calibration_multiplier"] as? Double ?: 1.0

        val scoreMap: Map<String, Double> = entityScorer.score(
            ids = rows.map { it.id },
            predictorName = predictorName,
            modelName = modelName,
            context = context,
        )

        rows.forEach { row ->
            row.score = (scoreMap[row.id] ?: 0.0) * calibrationMultiplier
            row.metadata["calibration_multiplier"] = calibrationMultiplier
        }
    }
}
```

`calibration_multiplier` is written to `row.metadata` so downstream steps (e.g. `DIVERSITY_RERANK`) can reference the pre-boost score if needed.

**Params contract:**
```
predictor_name         String   required  — Sibyl predictor
model_name             String   required  — Sibyl model
calibration_multiplier Double   optional  default 1.0
```

## MultiplierBoostStep (type = "MULTIPLIER_BOOST")

```kotlin
// libraries/common/.../ubp/vertical/steps/MultiplierBoostStep.kt
class MultiplierBoostStep : FeedRowRankingStep {
    override val stepType = "MULTIPLIER_BOOST"

    override suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: Map<String, Any>,
    ) {
        @Suppress("UNCHECKED_CAST")
        val multipliers = params["row_type_multipliers"] as Map<String, Double>

        rows.forEach { row ->
            val boost = multipliers[row.type.name] ?: 1.0
            row.score *= boost
        }
    }
}
```

**Params contract:**
```
row_type_multipliers  Map<String,Double>  required  — key is RowType.name, value is multiplier
```

Keys are `RowType.name` strings (e.g. `"STORE_CAROUSEL"`) — no `RowType` import needed in the params map. Rows with no matching key are unaffected (multiplier defaults to 1.0).

## DiversityRerankStep (type = "DIVERSITY_RERANK")

```kotlin
// libraries/common/.../ubp/vertical/steps/DiversityRerankStep.kt
class DiversityRerankStep : FeedRowRankingStep {
    override val stepType = "DIVERSITY_RERANK"

    override suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: Map<String, Any>,
    ) {
        val weight = params["weight"] as Double
        val maxPerType = params["max_per_type"] as? Int ?: Int.MAX_VALUE

        // MMR-style reranking: score' = (1-weight)*relevance + weight*diversity
        // diversity = 1 - (count of this type already selected / total selected)
        val selected = mutableListOf<FeedRow>()
        val typeCounts = mutableMapOf<RowType, Int>()
        val remaining = rows.toMutableList()

        rows.clear()
        while (remaining.isNotEmpty()) {
            val next = remaining.maxByOrNull { row ->
                val typeCount = typeCounts[row.type] ?: 0
                if (typeCount >= maxPerType) Double.MIN_VALUE
                else {
                    val relevance = row.score
                    val diversity = 1.0 - (typeCount.toDouble() / (selected.size + 1).coerceAtLeast(1))
                    (1 - weight) * relevance + weight * diversity
                }
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
weight       Double  required  — 0.0 = pure relevance, 1.0 = pure diversity
max_per_type Int     optional  — hard cap on same-type rows, default unlimited
```

## FixedPinningStep (type = "FIXED_PINNING")

```kotlin
// libraries/common/.../ubp/vertical/steps/FixedPinningStep.kt
class FixedPinningStep : FeedRowRankingStep {
    override val stepType = "FIXED_PINNING"

    override suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: Map<String, Any>,
    ) {
        @Suppress("UNCHECKED_CAST")
        val pins = params["pins"] as List<Map<String, Any>>

        // pins = [{ "row_type": "NV_CAROUSEL", "position": 0 }, ...]
        // Position is 0-indexed. Rows not in pins stay in their ranked order.

        val pinsByPosition = pins.associate { pin ->
            val position = pin["position"] as Int
            val rowType = RowType.valueOf(pin["row_type"] as String)
            position to rowType
        }

        val pinned = mutableMapOf<Int, FeedRow>()
        val unpinned = rows.toMutableList()

        pinsByPosition.forEach { (position, rowType) ->
            val row = unpinned.firstOrNull { it.type == rowType } ?: return@forEach
            unpinned.remove(row)
            pinned[position] = row
        }

        rows.clear()
        var unpinnedIdx = 0
        val totalSize = pinned.size + unpinned.size
        for (i in 0 until totalSize) {
            rows.add(pinned[i] ?: unpinned[unpinnedIdx++])
        }
    }
}
```

**Params contract:**
```
pins  List<Map<String,Any>>  required
  each pin: { "row_type": String (RowType.name), "position": Int (0-indexed) }
```

`FIXED_PINNING` uses `RowType` strings in params — no casting to `FeedRow` subtypes. This is the intended pattern for type-specific logic without downcasting.

## Files to Create

| File | Purpose |
|---|---|
| `libraries/common/.../ubp/vertical/FeedRowRankingStep.kt` | Interface |
| `libraries/common/.../ubp/vertical/steps/ModelScoringStep.kt` | Step implementation |
| `libraries/common/.../ubp/vertical/steps/MultiplierBoostStep.kt` | Step implementation |
| `libraries/common/.../ubp/vertical/steps/DiversityRerankStep.kt` | Step implementation |
| `libraries/common/.../ubp/vertical/steps/FixedPinningStep.kt` | Step implementation |

## Testing

Each step verifies: (1) effect on `row.score`, (2) params injected not DV-read, (3) no real `EntityScorer` — mock it.

```kotlin
@Test
fun `ModelScoringStep applies calibration multiplier to score`() {
    val scorer = mockk<EntityScorer>()
    coEvery { scorer.score(any(), "predictor", "model_v1", any()) } returns mapOf("r1" to 0.8)

    val step = ModelScoringStep(scorer)
    val rows = mutableListOf(fakeRow("r1", initialScore = 0.0))
    runBlocking {
        step.process(rows, fakeContext(), mapOf(
            "predictor_name" to "predictor",
            "model_name" to "model_v1",
            "calibration_multiplier" to 2.0,
        ))
    }
    assertEquals(1.6, rows[0].score, 0.001)
}

@Test
fun `ModelScoringStep writes calibration_multiplier to metadata`() {
    val scorer = mockk<EntityScorer>()
    coEvery { scorer.score(any(), any(), any(), any()) } returns mapOf("r1" to 0.5)

    val step = ModelScoringStep(scorer)
    val rows = mutableListOf(fakeRow("r1", initialScore = 0.0))
    runBlocking {
        step.process(rows, fakeContext(), mapOf(
            "predictor_name" to "p",
            "model_name" to "m",
            "calibration_multiplier" to 1.5,
        ))
    }
    assertEquals(1.5, rows[0].metadata["calibration_multiplier"])
}

@Test
fun `MultiplierBoostStep multiplies score by row type`() {
    val step = MultiplierBoostStep()
    val rows = mutableListOf(
        fakeRow("s1", type = RowType.STORE_CAROUSEL, initialScore = 0.5),
        fakeRow("n1", type = RowType.NV_CAROUSEL, initialScore = 0.5),
    )
    runBlocking {
        step.process(rows, fakeContext(), mapOf(
            "row_type_multipliers" to mapOf("STORE_CAROUSEL" to 2.0)
        ))
    }
    assertEquals(1.0, rows[0].score, 0.001)   // boosted
    assertEquals(0.5, rows[1].score, 0.001)   // unchanged
}

@Test
fun `DiversityRerankStep interleaves types when weight is high`() {
    val step = DiversityRerankStep()
    val rows = mutableListOf(
        fakeRow("s1", type = RowType.STORE_CAROUSEL, initialScore = 1.0),
        fakeRow("s2", type = RowType.STORE_CAROUSEL, initialScore = 0.9),
        fakeRow("n1", type = RowType.NV_CAROUSEL, initialScore = 0.5),
    )
    runBlocking {
        step.process(rows, fakeContext(), mapOf("weight" to 0.9))
    }
    // With high diversity weight, NV_CAROUSEL should not be last
    assertNotEquals(RowType.NV_CAROUSEL, rows[2].type)
}

@Test
fun `FixedPinningStep places NV_CAROUSEL at position 0`() {
    val rows = mutableListOf(
        fakeRow("s1", type = RowType.STORE_CAROUSEL),
        fakeRow("n1", type = RowType.NV_CAROUSEL),
        fakeRow("s2", type = RowType.STORE_CAROUSEL),
    )
    val step = FixedPinningStep()
    runBlocking {
        step.process(rows, fakeContext(), mapOf(
            "pins" to listOf(mapOf("row_type" to "NV_CAROUSEL", "position" to 0))
        ))
    }
    assertEquals(RowType.NV_CAROUSEL, rows[0].type)
    assertEquals(3, rows.size)
}

@Test
fun `FixedPinningStep preserves ranked order for unpinned rows`() {
    val rows = mutableListOf(
        fakeRow("s1", type = RowType.STORE_CAROUSEL),
        fakeRow("s2", type = RowType.STORE_CAROUSEL),
        fakeRow("n1", type = RowType.NV_CAROUSEL),
    )
    val step = FixedPinningStep()
    runBlocking {
        step.process(rows, fakeContext(), mapOf(
            "pins" to listOf(mapOf("row_type" to "NV_CAROUSEL", "position" to 2))
        ))
    }
    assertEquals("s1", rows[0].id)
    assertEquals("s2", rows[1].id)
    assertEquals("n1", rows[2].id)
}
```

## Value Function Mapping

These 4 steps implement the Phase 1 approximation of the UBP value function:

```
Full EV  = pImp(k)  ×   pAct(c)        ×   vAct(c)
Phase 1  =   1.0    ×  MODEL_SCORING   ×  MULTIPLIER_BOOST
```

- **`MODEL_SCORING`** computes `pAct(c)` — probability of user action given they see this content, from Sibyl CTR/CVR model
- **`MULTIPLIER_BOOST`** approximates `vAct(c)` — boost weights encode that "NV content at equal CTR is worth X× more to the business than organic"
- **`pImp(k)`** (probability user sees position k, decays with position) is **not implemented in Phase 1** — steps have no knowledge of final position during scoring. Comes in Phase 3.
- **Step order in config is load-bearing**: `MODEL_SCORING` must precede `MULTIPLIER_BOOST` since boost multiplies into the model score

## Scope Notes

**`FIXED_PINNING` is for MLE-configured experiment pins only.** It does NOT replace:
- `NonRankableHomepageOrderingUtil.updateSortOrderOfNVCarousel()` — post-checkout NV boosting (stays as post-ranking fixup)
- `updateSortOrderOfTasteOfDashPass()` — PAD=3 logic (stays, controlled by its own DV)
- `updateSortOrderOfMemberPricing()` — member pricing priority (stays, runtime feature flag)

**No ads in Phase 1.** `RowType` does not include `AD_CAROUSEL`. Ads remain post-ranking insertion via `maybeAddSponsoredCarousel()`. Phase 3+ vision only.

## Prerequisites

Part 2 (FeedRow interface + RowType enum) must be complete. `EntityScorer` is an existing interface — `ModelScoringStep` takes it by constructor injection. No new dependencies introduced.

## Done When

- All 4 step classes created and unit tested
- Each step's params contract is documented inline and in this file
- No step reads a DV key internally
- `ModelScoringStep` uses constructor-injected `EntityScorer` (no service locator)
- Tests pass without a running DV server or real scoring service
