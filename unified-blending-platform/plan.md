# Unified Blending Platform — Plan

## Status: Abstractions First → Engine → Traffic Management

> **Pivot (March 2026):** After stakeholder conversations (YZ, Dipali, Frank), the approach
> shifts from "build a full engine then prove it" to "ship good abstractions in feed-service,
> extract seams, build the engine on proven foundations." The MLE-facing contract is simplified.
> Per-layer traffic management is the #1 customer-facing deliverable.
>
> See `context/pivot-analysis.md` and `context/rfc-feedback.md` for full synthesis.

---

## Approach: Working Effectively with Legacy Code

The feed-service post-processing code is messy and poorly understood (confirmed by every
stakeholder). Rewriting it is too risky and too slow. Instead:

1. **Extract interfaces at seams** — FeedRow wraps 9+ carousel types behind one interface.
   RankingStep wraps hardcoded ranking methods behind one interface. The old code still works.
2. **Ship abstractions first** — Each abstraction is independently useful. FeedRow solves
   Frank's new-carousel-type problem. RankingStep makes ranking composable. Neither requires
   the full engine to deliver value.
3. **Build the engine on proven foundations** — Once FeedRow and RankingStep exist and are
   tested, the engine is just a loop over steps. Ship it when the abstractions are proven.
4. **Traffic router in parallel** — Per-layer traffic management doesn't depend on the engine.
   Build it concurrently.

This follows the "Scratch Refactoring" pattern from *Working Effectively with Legacy Code*:
understand by refactoring, formalize into real abstractions, ship incrementally.

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
│  LAYER 3: HORIZONTAL RANKING                             │
│  DefaultHomePageStoreRanker.rank()                       │
│  Ranks stores/items WITHIN each carousel by score        │
│  Currently: per-RankingType when-chain in                │
│    modifyLiteStoreCollection()                           │
│  Output: List<LiteStoreCollection> (stores re-ordered)   │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 4: VERTICAL RANKING                               │
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

```
reOrderGlobalEntitiesV2()
  ├── rankAndDedupeContent()              ← UBP replaces THIS path for UBP requests
  │     ├── rankAndMergeContent()
  │     │     ├── split pinned vs rankable
  │     │     └── score rankable via Sibyl → BlendingUtil → assign sortOrder
  │     ├── ads scoring + blending
  │     └── deduplication                 ← stays as-is
  │
  ├── post-ranking fixups                 ← stay as-is (business rules, not MLE experiments)
  │     ├── updateSortOrderOfNVCarousel()
  │     ├── updateSortOrderOfTasteOfDashPass()
  │     └── updateSortOrderOfMemberPricing()
  │
  └── NonRankableHomepageOrderingUtil     ← stays as-is (color bleed, spacing, presentation)
        .rankWithImmersivesV2()
```

---

## Stakeholder Pain Points (Why This Order)

| Stakeholder | Role | #1 Pain | UBP Component |
|---|---|---|---|
| **Frank Zhang** | NV BE | New carousel type onboarding — no abstraction | FeedRow interface + adapters |
| **Yu Zhang** | Manager | DV waterfall — experiment traffic management | Per-layer traffic router |
| **Dipali Ranjan** | MLE | Model experiment lifecycle — config mess | Simplified experiment declaration |

All agree: code is messy, no abstraction layer, adding anything requires deep HP knowledge.

---

## MLE-Facing Contract (Simplified)

MLEs provide an experiment declaration. UBP handles everything else.

```json
{
  "experiment_id": "nv_boost_v2",
  "layer": "vertical",
  "model_name": "vertical_intent_model_v3",
  "predictor_name": "FEED_RANKING_FW_SIBYL_PREDICTOR_NAME",
  "traffic_pct": 5,
  "params": {}
}
```

| Field | Required | Description |
|---|---|---|
| `experiment_id` | Yes | Unique experiment name |
| `layer` | Yes | `"vertical"` or `"horizontal"` |
| `model_name` | Yes | Sibyl model to use |
| `predictor_name` | No | Defaults to layer's default predictor |
| `traffic_pct` | Yes | % of traffic within the layer |
| `params` | No | Step-level param overrides (empty = model swap only) |

MLEs don't write pipeline configs or step sequences. That's internal.

---

## Internal Engine Architecture (HP-Owned)

### Naming

- `FeedRow` — vertical ranking unit (one carousel on the page)
- `RowItem` — horizontal ranking unit (one store/item within a carousel)
- `FeedRowRankingStep` / `RowItemRankingStep` — one step in the ranking pipeline
- `FeedRowRanker` / `RowItemRanker` — config-driven orchestrator

### Vertical step types

| Step type | What it does | Replaces in current code |
|---|---|---|
| `MODEL_SCORING` | Call Sibyl with predictor + features | `EntityRankerConfiguration.getScoreBundleWithWorkflowHelper()` |
| `MULTIPLIER_BOOST` | Calibration x intent x vertical boost weights | `BlendingUtil.blendBundle()` |
| `DIVERSITY_RERANK` | Greedy rerank penalizing non-Rx density | `BlendingUtil.rerankEntitiesWithDiversity()` |
| `POSITION_BOOSTING` | Deal multiplier + boost-by-position + NV unpin | `Boosting.kt` |
| `FIXED_PINNING` | MLE-configured position pins | `BoostingBundle.boosted()` |

### Config fragmentation this solves

```
What UBP puts in one place              Current location
──────────────────────────               ──────────────────────────────────────
predictor_name / model_name           →  StoreCollectionScorer.kt (hardcoded) + 3 separate DVs
calibration_config                    →  ranking/hp_vertical_blending_config.json
intent_scoring_config                 →  ranking/hp_vertical_blending_config.json
vertical_boost_weights                →  ranking/hp_vertical_blending_config.json
diversity params                      →  ranking/hp_vertical_blending_config.json
pinned carousel order                 →  carousels/pinned_carousel_ranking_order.json
boost allow list + DV                 →  boostByPositionCarouselIdAllowList + ENABLE_UR_BOOSTING DV
```

---

## N x M Experiment Model

Experiments are mutually exclusive WITHIN a layer, orthogonal ACROSS layers.

```
Layer: vertical    → Experiment A (5%), Experiment B (10%), Control (85%)
Layer: horizontal  → Experiment C (3%), Control (97%)
→ Any (vertical, horizontal) combination is valid
```

UBP traffic router assigns users via hash-based bucketing per layer:
`bucket = hash(consumer_id, layer_salt) % 1000`

No DV waterfall. No dummy DVs. No reserve segments. No priority lists.

---

## Implementation Steps (New Priority Order)

### Step 1: FeedRow interface + adapters

**Goal:** Single interface wrapping all carousel types. Immediately useful — new carousel types
need only one adapter class instead of touching 10+ files. Solves Frank's #1 pain.

**What to build:**
- `interface FeedRow` with `id`, `type: RowType`, `score` (mutable), `metadata`, `applyBackTo()`
- `RowType` enum: `STORE_CAROUSEL`, `ITEM_CAROUSEL`, `DEAL_CAROUSEL`, `STORE_COLLECTION`,
  `COLLECTION_V2`, `ITEM_COLLECTION`, `MAP_CAROUSEL`, `REELS_CAROUSEL`, `STORE_ENTITY`
- Adapters for all 9 types (each: `toFeedRow()` extension + `applyBackTo()`)
- Extend existing `ScorableEntityStoreCarousel` + `ScorableEntityItemCarousel` prototypes

**Key files:**
- New: `libraries/common/.../ubp/vertical/FeedRow.kt`
- New: `libraries/common/.../ubp/vertical/RowType.kt`
- New: `libraries/common/.../ubp/vertical/adapters/` (9 adapter classes)

**Why first:** This is a pure seam extraction. Zero risk to existing code. Immediately testable.
Any code that branches on carousel type can start using it.

---

### Step 2: FeedRowRankingStep interface + step extraction

**Goal:** Extract hardcoded ranking methods into registered, independently-testable steps.

| Step class | Step type | Wraps |
|---|---|---|
| `ModelScoringStep` | `MODEL_SCORING` | `entityScorer.score()` + Sibyl RPC |
| `MultiplierBoostStep` | `MULTIPLIER_BOOST` | `BlendingUtil.blendBundle()` |
| `DiversityRerankStep` | `DIVERSITY_RERANK` | `BlendingUtil.rerankEntitiesWithDiversity()` |
| `FixedPinningStep` | `FIXED_PINNING` | `BoostingBundle.boosted()` |

**Critical rule:** `params` injected from config. Steps do NOT read DV keys internally.

```kotlin
interface FeedRowRankingStep {
    val stepType: String
    val paramsClass: Class<out StepParams>
    suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: StepParams,
    )
}
```

**Key files:**
- New: `libraries/common/.../ubp/vertical/FeedRowRankingStep.kt`
- New: `libraries/common/.../ubp/vertical/steps/` (4 implementations)

**Why second:** Steps consume FeedRow (depends on Step 1). Extraction is the scratch-refactoring
approach: move logic into step classes, verify behavior is preserved, formalize.

---

### Step 3: Per-layer traffic router (parallel with Steps 1-2)

**Goal:** Replace DV waterfall with hash-based bucket partitioning per layer. Solves YZ's #1 pain.

**What to build:**
- `UbpExperimentRegistry` — runtime JSON listing active experiments per layer
- `UbpTrafficRouter` — hash-based assignment (`murmurHash(consumerId, layerSalt) % 1000`)
- Writes assignment to `experimentMap` for analytics pipeline compatibility
- Control config in separate HP-owned runtime JSON

```kotlin
class UbpTrafficRouter(private val registry: UbpExperimentRegistry) {
    fun assign(consumerId: Long, layer: String): ExperimentAssignment {
        val bucket = murmurHash(consumerId, layer) % 1000
        val experiments = registry.getExperiments(layer)
        for (exp in experiments) {
            if (bucket in exp.bucketStart..exp.bucketEnd) {
                return ExperimentAssignment(exp.id, exp.treatment, resolveConfig(exp))
            }
        }
        return ExperimentAssignment.CONTROL
    }
}
```

**Key files:**
- New: `libraries/common/.../ubp/traffic/UbpExperimentRegistry.kt`
- New: `libraries/common/.../ubp/traffic/UbpTrafficRouter.kt`

**Why parallel:** No dependency on FeedRow or steps. Can be built and tested independently.

---

### Step 4: FeedRowRanker engine + step registry

**Goal:** Config-driven orchestrator. Zero business logic. Pure dispatch.

```kotlin
class FeedRowRanker(
    private val stepRegistry: StepRegistry,
    private val traceEmitter: UbpTraceEmitter? = null,
) {
    suspend fun rank(
        rows: MutableList<FeedRow>,
        pipeline: ResolvedPipeline,
        context: RankingContext,
    ): List<FeedRow>
}
```

- Loops steps from resolved pipeline, dispatches to `stepRegistry[stepConfig.type]`
- Unknown step type → skip + log warning (never crash)
- Auto-trace after each step when `emitTrace = true`

**Key files:**
- New: `libraries/common/.../ubp/vertical/FeedRowRanker.kt`
- New: `libraries/common/.../ubp/config/` (config data classes, control.json)

**Why fourth:** Depends on FeedRow (Step 1) and steps (Step 2). Simple once foundations exist.

---

### Step 5: Wire engine + traffic router into feed-service

**Goal:** Route requests via traffic router to UBP engine. Zero impact to non-UBP traffic.

**Vertical incision:** inside `reOrderGlobalEntitiesV2()`, before `rankAndDedupeContent()`:
```kotlin
val assignment = ubpTrafficRouter.assign(consumerId, "vertical")
if (assignment != ExperimentAssignment.CONTROL) {
    val rows = toFeedRows(layout)
    val ranked = feedRowRanker.rank(rows, assignment.pipeline, context)
    applyRankedRows(ranked, layout)
} else {
    rankAndDedupeContent(...)
}
```

**Horizontal incision:** inside `DefaultHomePageStoreRanker.rank()`, wrapping
`modifyLiteStoreCollection()`.

**Key files:**
- `pipelines/homepage/.../DefaultHomePagePostProcessor.kt`
- `pipelines/homepage/.../DefaultHomePageStoreRanker.kt`

---

### Step 6: Standardized tracing

**Goal:** Auto-trace per step. Score snapshots queryable in Snowflake.

- Standard event: `{ row_id, step_id, score_before, score_after, request_id, experiment_id }`
- Engine emits when `output_config.emit_trace = true`
- New proto: `ubp_feed_row_ranking_trace.proto` in services-protobuf

---

### Step 7: Shadow validation + contract assembly

**Goal:** Prove UBP engine produces identical output to old path.

- `UbpContractAssembler` observes prod decisions, assembles equivalent UBP config
- Shadow: run both paths, compare sort order, log divergences
- Target: `divergence_count = 0` before any real traffic migration

---

## Migration Path

| Stage | What | Traffic |
|---|---|---|
| 0 | Ship abstractions (FeedRow, steps) — no traffic impact | 0% |
| 1 | Contract assembly — shadow observe prod decisions | 0% |
| 2 | Shadow validate — UBP engine vs old path comparison | 0% |
| 3 | Control arm — small % real traffic to UBP control | 1-5% |
| 4 | First experiment — MLE declares experiment via Contract 0 | MLE-chosen % |
| 5 | Retire — old experiments end, remove old code paths | Gradual |

---

## New Carousel Type Onboarding Process (Frank's Pain)

Once FeedRow exists, the onboarding process for a new carousel type becomes:

```
1. Write FeedRowAdapter for the new type (1 class)
2. Stage 1 — Pin: configure FIXED_PINNING step with guaranteed position
3. Stage 2 — Explore: enable UCB_EXPLORATION step for the new type
4. Stage 3 — Organic: remove pin, let MODEL_SCORING rank naturally
```

Each stage is a config change, not a code change. The thresholds for transitioning
between stages can be defined in config (min impressions, min MAB trials, etc.).

---

## Key Files Reference

| Layer | Key File | What it does |
|---|---|---|
| Vertical entry | `DefaultHomePagePostProcessor.kt` | `reOrderGlobalEntitiesV2()` — incision point |
| Vertical scoring | `EntityRankerConfiguration.kt` | Calls Sibyl + blending (replaced by FeedRowRanker) |
| Vertical blending | `VerticalBlending.kt` | Calibration x intent x boost weight multipliers |
| Vertical pinning | `Boosting.kt` | `enforceManualSortOrder` enforcement |
| Pinned order config | `PinnedCarouselUtil.kt` | Reads `pinned_carousel_ranking_order.json` |
| Post-ranking fixups | `NonRankableHomepageOrderingUtil.kt` | NV, PAD, gap rules |
| Blending config | `VerticalBlendingConfig.kt` | Existing blending params |
| Horizontal entry | `DefaultHomePageStoreRanker.kt` | `rank()` — incision point |
| Horizontal when-chain | `DefaultHomePageStoreRanker.kt` | `modifyLiteStoreCollection()` |
| Horizontal scoring | `StoreCollectionScorer.kt` | Sibyl RPC + feature construction |
| Experiment manager | `DiscoveryExperimentManager.kt` | DV manifest |
| Pipeline DAG | `DefaultHomePagePipeline.kt` | Job dependency graph |

---

## Progress Tracker

**Abstractions (ship first — independently valuable)**
- [ ] Step 1: `FeedRow` interface + adapters for all 9 carousel types
- [ ] Step 2: `FeedRowRankingStep` interface + 4 step implementations

**Traffic Management (parallel with abstractions)**
- [ ] Step 3: `UbpExperimentRegistry` + `UbpTrafficRouter` (hash-based bucketing)

**Engine (builds on abstractions)**
- [ ] Step 4: `FeedRowRanker` engine + step registry + config data classes
- [ ] Step 5: Wire engine + traffic router into PostProcessor + StoreRanker
- [ ] Step 6: Standardized tracing / `ubp_feed_row_ranking_trace.proto`
- [ ] Step 7: Shadow validation + contract assembly

**Horizontal (mirrors vertical)**
- [ ] `RowItem` interface + adapters
- [ ] `RowItemRankingStep` + step implementations
- [ ] `RowItemRanker` with `row_overrides` dispatch
- [ ] Wire into `DefaultHomePageStoreRanker`

**Post-POC**
- [ ] `CalibrationStep` (`PIECEWISE`, `ISOTONIC`)
- [ ] `ValueFunctionStep` (pImp x pAct x vAct)
- [ ] `AdsFeedRow` adapter + fair ads/organic competition
