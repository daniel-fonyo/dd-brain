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
│  LAYER 3: HORIZONTAL RANKING  ← Phase 2                  │
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

## MLE Contract (The Foundation)

The contract is what MLEs write. The implementation is what makes the contract executable.

**Canonical contract doc:** `context/mle-vertical-contract.md`
**Full Kotlin schema + engine spec:** `feed-service/ubp-experiment-contract.md`

### What MLEs write

One JSON file per experiment. Two modes:

**Mode 1 — param override (common case):**
Inherit the full control pipeline, change one or more step params via `"extends": "control"` + `"step_params"`.

**Mode 2 — full pipeline declaration:**
Declare the entire `vertical_pipeline.steps` array when the step sequence itself changes.

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
1. No Component interface — 9+ typed carousel classes with no shared score interface
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

### Step 2: `VerticalComponent` interface + adapters

**Goal:** Single interface wrapping all 9 carousel types so processors never branch on type.

**What to build:**
- `interface VerticalComponent` with `id`, `type` (enum), `score` (mutable), `metadata`, `recordTrace()`
- Adapters for all 9 types: `StoreCarouselComponent`, `ItemCarouselComponent`, `CollectionV2Component`, `DealCarouselComponent`, `StoreCollectionComponent`, `ItemCollectionComponent`, `MapCarouselComponent`, `ReelsCarouselComponent`, `StoreEntityComponent`
- Extend existing `ScorableEntityStoreCarousel` + `ScorableEntityItemCarousel` prototypes
- Each adapter: `toComponent()` extension + `applyBackTo(domainObject)` for writing score back

**Key files:**
- Extend: `libraries/common/.../scoring/ScorableEntity*.kt`
- New: `libraries/common/.../ubp/vertical/VerticalComponent.kt`
- New: `libraries/common/.../ubp/vertical/ComponentType.kt`

---

### Step 3: `VerticalProcessor` interface + 4 processor implementations

**Goal:** 4 hardcoded steps become registered, independently-testable processors.

| Processor class | Step type | Wraps |
|---|---|---|
| `ModelScoringProcessor` | `MODEL_SCORING` | `entityScorer.score()` + Sibyl RPC — reads `PredictorConfig` from params |
| `MultiplierBoostProcessor` | `MULTIPLIER_BOOST` | `BlendingUtil.blendBundle()` — params injected, no internal DV reads |
| `DiversityRerankProcessor` | `DIVERSITY_RERANK` | `BlendingUtil.rerankEntitiesWithDiversity()` |
| `FixedPinningProcessor` | `FIXED_PINNING` | `BoostingBundle.boosted()` + all `NonRankableHomepageOrderingUtil` fixups |

**Critical design rule:** `params` injected from config at call time. Processors do NOT read DV keys internally.

```kotlin
interface VerticalProcessor {
    val type: String
    suspend fun process(
        components: MutableList<VerticalComponent>,
        context: RankingContext,
        params: Map<String, Any>,
    )
}
```

**Key files:**
- New: `libraries/common/.../ubp/vertical/VerticalProcessor.kt`
- New: `libraries/common/.../ubp/vertical/processors/` (4 implementations)

---

### Step 4: `VerticalRankingEngine` with processor registry + auto-trace

**Goal:** Config-driven orchestrator. Zero business logic. Pure dispatch.

```kotlin
class VerticalRankingEngine(
    private val processorRegistry: Map<String, VerticalProcessor>,
    private val ubpRuntimeUtil: UbpRuntimeUtil,
) {
    suspend fun rank(
        components: MutableList<VerticalComponent>,
        experimentId: String,
        context: RankingContext,
    ): List<VerticalComponent>
}
```

- Resolves config via `ubpRuntimeUtil.resolveConfig(experimentId, context.experimentMap)`
- Loops steps, dispatches to `processorRegistry[step.type]`
- Unknown step type → skip + log warning (never crash)
- Auto-trace: `component.recordTrace("${step.id}.score", score)` after each step when `emitTrace = true`

**Key files:**
- New: `libraries/common/.../ubp/vertical/VerticalRankingEngine.kt`

---

### Step 5: Wire into `DefaultHomePagePostProcessor`

**Goal:** Route requests with a `ubp_hp_vertical_*` DV to the new engine. Zero impact to non-UBP traffic.

**What to build:**
- In `reOrderGlobalEntitiesV2()`: check `experimentMap` for any active `ubp_hp_vertical_*` DV key
- If UBP active:
  1. Convert carousels → `VerticalComponent`s via adapters
  2. Call `VerticalRankingEngine.rank(components, experimentId, context)`
  3. Write sorted scores back via `applyBackTo()` → sets `sortOrder`
  4. Skip existing `EntityRankerConfiguration.rank()` path for this request
- If not: existing code path unchanged
- Post-ranking fixups (Stage 7) → migrate to declared `FIXED_PINNING` steps in `control.json`

**Key files:**
- `pipelines/homepage/.../DefaultHomePagePostProcessor.kt`
- `pipelines/homepage/.../HomePagePostProcessorV2.kt`

---

### Step 6: Standardized tracing

**Goal:** Every step auto-traces. Stage-wise score snapshots queryable in Snowflake.

**What to build:**
- Standard trace event: `{ component_id, step_id, score_before, score_after, request_id, experiment_id }`
- Engine emits when `output_config.emit_trace = true`
- Replaces manual `IguazuEventUtil` calls per processor
- New proto: `ubp_vertical_ranking_trace.proto` in services-protobuf

**Key files:**
- New: `services-protobuf/protos/iguazu/events/ubp_vertical_ranking_trace.proto`
- New: `libraries/common/.../ubp/trace/UbpTraceEmitter.kt`

---

## Migration Path

```
Phase 1: Deploy UBP engine alongside existing pipeline (feature-flagged via DV)
  → Existing DVs → existing code path (unchanged)
  → New ubp_* DVs → new engine path
  → Validate: control arm produces same results as existing pipeline

Phase 2: Start first new experiment on new contract (not a migration of existing)
  → MLE authors first UnifiedExperimentConfig JSON
  → Iterate on treatments without code changes

Phase 3: Retire old DVs one by one
  → Remove branches from regressionContext() when-chain
  → Remove branches from modifyLiteStoreCollection() when-chain
  → Each removal independently shippable
```

---

## Phase 2 (After Vertical is Live): Horizontal Blending Platform

Same pattern applied to Layer 3:
- `HorizontalComponent` interface (wraps `DiscoveryStore`, `StoreEntity`, `LiteStoreCollection`)
- `HorizontalProcessor` interface with `carousel_overrides` support per `RankingType`
- `HorizontalRankingEngine`
- `BusinessRulesSortProcessor` replaces `modifyLiteStoreCollection()` when-chain
- `OrderHistoryRerankProcessor` makes S1/S2/S3 observable for the first time

---

## Phase 3+: Value Function, Calibration, Ads Integration

- `CalibrationService` (`PIECEWISE`, `ISOTONIC`) for cross-type score comparability
- `ValueFunctionProcessor` (pImp × pAct × vAct)
- `AdsStoreComponent` adapter + fair organic/ads competition on the same scale

---

## Key Files Reference

| Layer | Key File | What it does |
|---|---|---|
| Vertical entry | `DefaultHomePagePostProcessor.kt` | `reOrderGlobalEntitiesV2()` — 7-stage pipeline |
| Vertical scoring | `EntityRankerConfiguration.kt` | Calls Sibyl + blending (replaced by engine in UBP path) |
| Vertical blending | `VerticalBlending.kt` | Calibration × intent × boost weight multipliers |
| Vertical pinning | `Boosting.kt` | `enforceManualSortOrder` enforcement |
| Pinned order config | `PinnedCarouselUtil.kt` | Reads `pinned_carousel_ranking_order.json` |
| Post-ranking fixups | `NonRankableHomepageOrderingUtil.kt` | NV, PAD, gap rules |
| Blending config | `VerticalBlendingConfig.kt` | Existing blending params (absorbed into `UnifiedExperimentConfig`) |
| Horizontal entry | `DefaultHomePageStoreRanker.kt` | `modifyLiteStoreCollection()` per carousel |
| Horizontal scoring | `StoreCollectionScorer.kt` | Sibyl RPC + feature construction |
| Experiment manager | `DiscoveryExperimentManager.kt` | DV manifest |
| Pipeline DAG | `DefaultHomePagePipeline.kt` | Job dependency graph |

---

## Progress Tracker

**Phase 1: Vertical Blending**
- [ ] Step 1: `UnifiedExperimentConfig` data classes + `UbpRuntimeUtil` (config loading + `extends` merge)
- [ ] Step 2: `VerticalComponent` interface + adapters for all 9 carousel types
- [ ] Step 3: `VerticalProcessor` interface + 4 processor implementations
- [ ] Step 4: `VerticalRankingEngine` with processor registry + auto-trace
- [ ] Step 5: Wire into `DefaultHomePagePostProcessor` (behind UBP DV)
- [ ] Step 6: Standardized tracing / `ubp_vertical_ranking_trace.proto`

**Phase 2: Horizontal Blending**
- [ ] `HorizontalComponent` interface
- [ ] `HorizontalProcessor` + `BusinessRulesSortProcessor` + `OrderHistoryRerankProcessor`
- [ ] `HorizontalRankingEngine` with `carousel_overrides` dispatch
- [ ] Wire into `DefaultHomePageStoreRanker`

**Phase 3+: Value Function, Calibration, Ads**
- [ ] `CalibrationService` (`PIECEWISE`, `ISOTONIC`)
- [ ] `ValueFunctionProcessor` (pImp × pAct × vAct)
- [ ] `AdsStoreComponent` adapter + fair ads/organic competition
