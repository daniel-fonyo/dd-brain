# Unified Blending Platform — Plan

## Status: Phase 1 — Vertical Blending (Active)

---

## The Pipeline (4 Layers)

```
REQUEST
   │
   ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 1: RETRIEVAL                                      │
│  DefaultHomePageStoreContentFetching.fetch()             │
│  Fetches stores, items, campaigns from upstream services │
│  Output: HomePageStoreDiscoveryResponse                  │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 2: GROUPING                                       │
│  DefaultHomePageStoreContentGrouper.group()              │
│  Buckets candidates into named carousels via:            │
│    CampaignCarouselService (campaign-based)               │
│    ProgrammaticCarouselService (ML taste-based)          │
│    DealService, StoreListService                         │
│  Output: List<LiteStoreCollection>                       │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 3: HORIZONTAL RANKING  ← Phase 1.5                │
│  DefaultHomePageStoreRanker.rank()                       │
│  Ranks stores/items WITHIN each carousel by score        │
│  Currently: per-RankingType when-chain in                │
│    modifyLiteStoreCollection()                           │
│  Output: List<LiteStoreCollection> (stores re-ordered)   │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 4: VERTICAL RANKING    ← Phase 1 focus            │
│  DefaultHomePagePostProcessor.reOrderGlobalEntitiesV2()  │
│  Ranks carousels for their vertical page position via:   │
│    Universal Ranker (Sibyl) scoring                      │
│    BlendingUtil (calibration x intent x boost weights)   │
│    PinnedCarouselUtil (fixed-position carousels)         │
│    BoostingBundle (enforceManualSortOrder enforcement)   │
│    Post-ranking fixups (NV, PAD, member pricing, gaps)  │
│  Output: HomePageStoreLayoutOutputElements (sortOrder set)│
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
                  SERIALIZE → CLIENT
```

---

## What `reOrderGlobalEntitiesV2()` Actually Does

Important: this method does more than vertical ranking. Phase 1 only replaces one part of it.

```
reOrderGlobalEntitiesV2()
  ├── rankAndDedupeContent()              ← Phase 1 replaces THIS path for UBP requests
  │     ├── rankAndMergeContent()
  │     │     ├── split pinned vs rankable
  │     │     └── score rankable via Sibyl → BlendingUtil → assign sortOrder
  │     ├── ads scoring + blending        ← ads blend as RowItems in Phase 1.5 (horizontal)
  │     └── deduplication                 ← stays as-is
  │
  ├── post-ranking fixups                 ← stay as-is (business rules, not MLE experiments)
  │     ├── updateSortOrderOfNVCarousel()      NV post-checkout pin — context-driven, not config
  │     ├── updateSortOrderOfTasteOfDashPass() PAD=3 — owned by its own DV
  │     └── updateSortOrderOfMemberPricing()   feature flag, runtime ID list
  │
  └── NonRankableHomepageOrderingUtil     ← stays as-is (color bleed, spacing, presentation)
        .rankWithImmersivesV2()
```

Phase 1 UBP replaces `rankAndMergeContent()` for flagged requests. Everything else in the
method is unchanged.

---

## Naming

Types use `FeedRow` (vertical unit) and `RowItem` (horizontal unit). See `context/ubp-contracts.md`
for the full interface definitions.

Old names in existing code (renamed during implementation):
- `VerticalComponent` → `FeedRow`
- `HorizontalComponent` → `RowItem`
- `VerticalProcessor` → `FeedRowRankingStep`
- `VerticalRankingEngine` → `FeedRowRanker`

---

## MLE Contract (The Foundation)

The contract is what MLEs write. The implementation is what makes the contract executable.

**Canonical contract doc:** `context/ubp-contracts.md`

### What MLEs write

One JSON file per experiment. Two modes:

**Mode 1 — param override (common case):**
Inherit the full control pipeline, change one or more step params via `"extends": "control"` + `"step_params"`.

**Mode 2 — full pipeline declaration:**
Declare the entire `row_pipeline.steps` array when the step sequence itself changes.

### Vertical step types (Phase 1)

| Step type | What it does | Replaces in current code |
|---|---|---|
| `MODEL_SCORING` | Call Sibyl with MLE-declared predictor + features | `EntityRankerConfiguration.getScoreBundleWithWorkflowHelper()` |
| `MULTIPLIER_BOOST` | Calibration × intent × vertical boost weight multipliers | `BlendingUtil.blendBundle()` in `VerticalBlending.kt` |
| `DIVERSITY_RERANK` | Greedy rerank penalizing non-Rx density | `BlendingUtil.rerankEntitiesWithDiversity()` in `VerticalBlending.kt` |
| `FIXED_PINNING` | Pin a carousel to a specific page position | `BoostingBundle.boosted()` + `NonRankableHomepageOrderingUtil` post-ranking fixups |

### Config fragmentation this solves

```
What UBP puts in one JSON        Current location
────────────────────────         ──────────────────────────────────────
predictor_name / model_name   →  StoreCollectionScorer.kt (hardcoded) + 3 separate DVs
calibration_config            →  ranking/hp_vertical_blending_config.json
intent_scoring_config         →  ranking/hp_vertical_blending_config.json
vertical_boost_weights        →  ranking/hp_vertical_blending_config.json
diversity params              →  ranking/hp_vertical_blending_config.json
pinned carousel order         →  carousels/pinned_carousel_ranking_order.json
boost allow list + DV         →  boostByPositionCarouselIdAllowList + ENABLE_UR_BOOSTING DV
NV post-checkout pins         →  CarouselsRuntimeUtil (code)
PAD position                  →  PAD_CAROUSEL_SPOTLIGHT_V2 DV (hardcoded pos=3 in code)
```

---

## N × M Experiment Model

```
N experiments on horizontal ranking (Layer 3)
M experiments on vertical ranking (Layer 4)

DV "ubp_hp_horizontal_v2" → "treatment_fw"       (horizontal assignment)
DV "ubp_hp_vertical_v3"   → "intent_model__v3"   (vertical assignment)

Result: N × M valid cells, independent per layer.
```

---

## What the Current Code Gets Wrong (Root Causes)

**Vertical layer (Layer 4):**
1. No FeedRow interface — 9+ typed carousel classes with no shared score interface
2. No step registry — 4 hardcoded private method calls in `EntityRankerConfiguration.rank()`
3. Config fragmented across 10+ locations (3 runtime JSON files + 5+ DV keys + hardcoded constants)
4. Post-ranking fixups outside the DAG — run silently after pipeline, no traceability
5. Manual, inconsistent tracing — IguazuEventUtil called manually per-step, no standard schema

**Horizontal layer (Layer 3):**
1. RankingType god objects — each carousel type carries its own ranking logic baked in
2. S1/S2/S3 not observable — `rerankDecoratedEntities()` runs silently after Iguazu fires
3. Same config fragmentation as vertical

---

## Phase 1: Vertical Blending Platform — Implementation Steps

### Step 1: `UnifiedExperimentConfig` data classes + `UbpRuntimeUtil`

**Goal:** Load the MLE contract from Runtime JSON and resolve the active config.

**What to build:**
- Data classes: `UnifiedExperimentConfig`, `ValueFunctionConfig`, `PredictorConfig`, `FeatureSpec`, `PipelineConfig`, `StepConfig`, `CalibrationSpec`, `ConstraintConfig`, `OutputConfig`
- `UbpRuntimeUtil.resolveConfig(experimentId, experimentMap)` — reads Runtime JSON, falls back to `"control"` on missing key or parse error
- Support `"extends": "control"` + `"step_params"` shallow merge in the resolver
- Hot-reloadable via `DeserializedRuntimeCache` (5-minute TTL)
- Convention: DV label = `"ubp_{experiment_id}"`, JSON = `"ubp/experiments/{experiment_id}.json"`

**Key files:**
- New: `libraries/common/.../ubp/config/UnifiedExperimentConfig.kt`
- New: `libraries/common/.../ubp/config/UbpRuntimeUtil.kt`
- New: `ubp/experiments/control.json` (runtime JSON baseline)

---

### Step 2: `FeedRow` interface + adapters

**Goal:** Single interface wrapping all 9 carousel types so steps never branch on type.

**What to build:**
- `interface FeedRow` with `id`, `type: RowType` (enum), `score` (mutable), `metadata`, `recordTrace()`
- Adapters for all 9 types: `StoreCarouselRow`, `ItemCarouselRow`, `CollectionV2Row`, `DealCarouselRow`, `StoreCollectionRow`, `ItemCollectionRow`, `MapCarouselRow`, `ReelsCarouselRow`, `StoreEntityRow`
- Extend existing `ScorableEntityStoreCarousel` + `ScorableEntityItemCarousel` prototypes
- Each adapter: `toFeedRow()` extension + `applyBackTo()` for writing score back

**Key files:**
- Extend: `libraries/common/.../scoring/ScorableEntity*.kt`
- New: `libraries/common/.../ubp/vertical/FeedRow.kt`
- New: `libraries/common/.../ubp/vertical/RowType.kt`

---

### Step 3: `FeedRowRankingStep` interface + 4 step implementations

**Goal:** 4 hardcoded stages become registered, independently-testable steps.

| Step class | Step type | Wraps |
|---|---|---|
| `ModelScoringStep` | `MODEL_SCORING` | `entityScorer.score()` + Sibyl RPC — reads `PredictorConfig` from params |
| `MultiplierBoostStep` | `MULTIPLIER_BOOST` | `BlendingUtil.blendBundle()` — params injected, no internal DV reads |
| `DiversityRerankStep` | `DIVERSITY_RERANK` | `BlendingUtil.rerankEntitiesWithDiversity()` |
| `FixedPinningStep` | `FIXED_PINNING` | `BoostingBundle.boosted()` + all `NonRankableHomepageOrderingUtil` fixups |

**Critical design rule:** `params` injected from config at call time. Steps do NOT read DV keys internally.

```kotlin
interface FeedRowRankingStep {
    val type: String
    suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: Map<String, Any>,
    )
}
```

**Key files:**
- New: `libraries/common/.../ubp/vertical/FeedRowRankingStep.kt`
- New: `libraries/common/.../ubp/vertical/steps/` (4 implementations)

---

### Step 4: `FeedRowRanker` with step registry + auto-trace

**Goal:** Config-driven orchestrator. Zero business logic. Pure dispatch.

```kotlin
class FeedRowRanker(
    private val stepRegistry: Map<String, FeedRowRankingStep>,
    private val ubpRuntimeUtil: UbpRuntimeUtil,
) {
    suspend fun rank(
        rows: MutableList<FeedRow>,
        experimentId: String,
        context: RankingContext,
    ): List<FeedRow>
}
```

- Resolves config via `ubpRuntimeUtil.resolveConfig(experimentId, context.experimentMap)`
- Loops steps, dispatches to `stepRegistry[step.type]`
- Unknown step type → skip + log warning (never crash)
- Auto-trace: `row.recordTrace("${step.id}.score", score)` after each step when `emitTrace = true`

**Key files:**
- New: `libraries/common/.../ubp/vertical/FeedRowRanker.kt`

---

### Step 5: Wire into `DefaultHomePagePostProcessor`

**Goal:** Route requests with a `ubp_hp_vertical_*` DV to the new engine. Zero impact to non-UBP traffic.

**Incision point:** inside `reOrderGlobalEntitiesV2()`, before `rankAndDedupeContent()`:

```kotlin
val ubpExperimentId = context.experimentMap.keys
    .firstOrNull { it.startsWith("ubp_hp_vertical_") }

return if (ubpExperimentId != null) {
    val rows = toFeedRows(layout)
    val ranked = feedRowRanker.rank(rows, ubpExperimentId, context)
    applyRankedRows(ranked, layout)
} else {
    rankAndDedupeContent(...)
    NonRankableHomepageOrderingUtil.rankWithImmersivesV2(...)
}
```

Post-ranking fixups (NV pin, PAD, member pricing) → declared `FIXED_PINNING` steps in `control.json`.

**Key files:**
- `pipelines/homepage/.../DefaultHomePagePostProcessor.kt`

---

### Step 6: Standardized tracing

**Goal:** Every step auto-traces. Stage-wise score snapshots queryable in Snowflake.

**What to build:**
- Standard trace event: `{ row_id, step_id, score_before, score_after, request_id, experiment_id }`
- Engine emits when `output_config.emit_trace = true`
- Replaces manual `IguazuEventUtil` calls per step
- New proto: `ubp_feed_row_ranking_trace.proto` in services-protobuf

**Key files:**
- New: `services-protobuf/protos/iguazu/events/ubp_feed_row_ranking_trace.proto`
- New: `libraries/common/.../ubp/trace/UbpTraceEmitter.kt`

---

## Migration Path

### Stage 0: Deploy engine (0% real traffic)

UBP code ships but no `ubp_*` DV key exists. 100% of traffic takes the old code path.

### Stage 1: Shadow validation

Run both paths on every request. Always return the OLD result to user. Zero user impact.

```
shadow arm:
  1. Run old path  → result_old
  2. Run FeedRowRanker with control.json → result_new
  3. Return result_old to user
  4. Emit both sort orders to Snowflake
  5. Query divergence: WHERE result_old != result_new
```

Iterate on `control.json` until divergence is below threshold (target: >99% match on sort order).

**Seeding `control.json` from existing prod config — don't hand-author:**

For vertical, most params already live in `hp_vertical_blending_config.json`. Bootstrap by reading
it at init time:

```kotlin
fun buildControlConfig(): UnifiedExperimentConfig {
    val blendingConfig = P13nRuntimeUtil.getVerticalBlendingConfig()
    // calibration_entries, intent_scoring_config, boost_weights, diversity params
    // come directly from the existing runtime file — no drift risk

    return UnifiedExperimentConfig(
        rowPipeline = PipelineConfig(steps = listOf(
            StepConfig("score",     "MODEL_SCORING",    mapOf("predictor_ref" to "p_act")),
            StepConfig("blend",     "MULTIPLIER_BOOST", blendingConfig.toMultiplierParams()),
            StepConfig("diversity", "DIVERSITY_RERANK", blendingConfig.toDiversityParams()),
            StepConfig("pin",       "FIXED_PINNING",    buildPinningRules()),
        ))
    )
}

// Hardcoded fixups — values are known and stable
fun buildPinningRules() = mapOf("rules" to listOf(
    mapOf("row_id" to "pad",            "position" to 3, "hard_pin" to true),
    mapOf("row_id" to "member_pricing", "position" to 0, "hard_pin" to true),
))
```

Shadow comparison catches anything the bootstrap misses.

### Stage 2: First real traffic on UBP control arm

Route a small % of real traffic to UBP control arm. Identical page layout to old path.
Confirms no production issues.

### Stage 3: First experiment on UBP

MLE authors treatment JSON, no code change required. This is the payoff.

### Stage 4: Retire old DVs

When old experiments end naturally, don't rebuild them on old infra. Remove old code branches
one by one — each independently shippable.

---

## Phase 1.5: Horizontal Blending Platform (immediately after Phase 1)

Phase 1 (vertical) + Phase 1.5 (horizontal) together are the "aha moment": both layers of the
feed are config-driven, and ads blend natively within carousels as RowItems.

**Why horizontal is the ads story:** Ads blend at the Store Ranker level, not the vertical level.
Ad candidates and organic stores compete as `RowItem` objects within the same carousel ranking
step. The `MODEL_SCORING` step scores both together; the ranked list is the blended result.
There is no separate ads insertion pass. See `context/Homepage Ads Blending.md` for full context.

Same pattern applied to Layer 3. Key difference from vertical: horizontal runs once per carousel
per request (parallel `awaitAll()`), not once per request total.

**Incision point:** `DefaultHomePageStoreRanker.rank()`, wrapping each `modifyLiteStoreCollection()` call:

```kotlin
return scoredCollections.map { collection ->
    CoroutineScope(...).async {
        val ubpExperimentId = context.experimentMap.keys
            .firstOrNull { it.startsWith("ubp_hp_horizontal_") }

        if (ubpExperimentId != null) {
            rowItemRanker.rank(collection, ubpExperimentId, context)
        } else {
            modifyLiteStoreCollection(context, collection, storesById, metadata, ...)
        }
    }
}.awaitAll()
```

**Seeding horizontal `control.json` — map the when-chain once:**

No structured runtime JSON exists for horizontal. Map `modifyLiteStoreCollection()` when-chain
to `row_overrides` once, then it's static:

```
RankingType.CAROUSEL, NEW_VERTICAL_CAROUSEL, ...  → default (MODEL_SCORING + BUSINESS_RULES_SORT)
RankingType.PICKUP_STORE_FEED, MAP_CAROUSEL        → row_overrides: score.skip=true, rules.sort_by=DISTANCE
RankingType.BOOKMARKS                              → row_overrides: score.skip=true, rules.sort_by=SAVELIST_ORDER
RankingType.WHOLE_ORDER_REORDER                    → row_overrides: ORDER_HISTORY_RERANK.enabled=true
RankingType.NOOP                                   → row_overrides: steps=[]
```

**What to build:**
- `RowItem` interface (wraps `DiscoveryStore` / `StoreEntity`) with `id`, `score`, `applyBackTo()`
- `RowItemRankingStep` interface
- `RowItemRanker`
- `BusinessRulesSortStep` replaces `modifyLiteStoreCollection()` when-chain
- `OrderHistoryRerankStep` makes S1/S2/S3 observable for the first time

---

## Phase 3+: Value Function, Calibration, Ads Integration

- `CalibrationStep` (`PIECEWISE`, `ISOTONIC`) for cross-type score comparability
- `ValueFunctionStep` (pImp × pAct × vAct)
- `AdsFeedRow` adapter + fair organic/ads competition on the same scale

---

## Key Files Reference

| Layer | Key File | What it does |
|---|---|---|
| Vertical entry | `DefaultHomePagePostProcessor.kt` | `reOrderGlobalEntitiesV2()` — Phase 1 incision point |
| Vertical scoring | `EntityRankerConfiguration.kt` | Calls Sibyl + blending (replaced by `FeedRowRanker` in UBP path) |
| Vertical blending | `VerticalBlending.kt` | Calibration × intent × boost weight multipliers |
| Vertical pinning | `Boosting.kt` | `enforceManualSortOrder` enforcement |
| Pinned order config | `PinnedCarouselUtil.kt` | Reads `pinned_carousel_ranking_order.json` |
| Post-ranking fixups | `NonRankableHomepageOrderingUtil.kt` | NV, PAD, gap rules |
| Blending config | `VerticalBlendingConfig.kt` | Existing blending params — bootstraps `control.json` |
| Horizontal entry | `DefaultHomePageStoreRanker.kt` | `rank()` — Phase 2 incision point |
| Horizontal when-chain | `DefaultHomePageStoreRanker.kt` | `modifyLiteStoreCollection()` — maps to `row_overrides` |
| Horizontal scoring | `StoreCollectionScorer.kt` | Sibyl RPC + feature construction |
| Experiment manager | `DiscoveryExperimentManager.kt` | DV manifest |
| Pipeline DAG | `DefaultHomePagePipeline.kt` | Job dependency graph |

---

## Progress Tracker

**Phase 1: Vertical Blending**
- [ ] Step 1: `UnifiedExperimentConfig` data classes + `UbpRuntimeUtil` (config loading + `extends` merge)
- [ ] Step 2: `FeedRow` interface + adapters for all 9 carousel types
- [ ] Step 3: `FeedRowRankingStep` interface + 4 step implementations
- [ ] Step 4: `FeedRowRanker` with step registry + auto-trace
- [ ] Step 5: Wire into `DefaultHomePagePostProcessor` (behind UBP DV)
- [ ] Step 6: Standardized tracing / `ubp_feed_row_ranking_trace.proto`

**Phase 2: Horizontal Blending**
- [ ] `RowItem` interface
- [ ] `RowItemRankingStep` + `BusinessRulesSortStep` + `OrderHistoryRerankStep`
- [ ] `RowItemRanker` with `row_overrides` dispatch
- [ ] Wire into `DefaultHomePageStoreRanker`

**Phase 3+: Value Function, Calibration, Ads**
- [ ] `CalibrationStep` (`PIECEWISE`, `ISOTONIC`)
- [ ] `ValueFunctionStep` (pImp × pAct × vAct)
- [ ] `AdsFeedRow` adapter + fair ads/organic competition
