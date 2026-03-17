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
│  LAYER 3: HORIZONTAL RANKING  ← UBP scope                │
│  DefaultHomePageStoreRanker.rank()                       │
│  Ranks stores/items WITHIN each carousel by score        │
│  Currently: per-RankingType when-chain in                │
│    modifyLiteStoreCollection()                           │
│  Output: List<LiteStoreCollection> (stores re-ordered)   │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 4: VERTICAL RANKING    ← UBP scope, focus first   │
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

## N × M Experiment Model (Verified Against Codebase)

The N×M model is correct and already supported by the DV infrastructure:

```
N experiments on horizontal ranking (Layer 3)
M experiments on vertical ranking (Layer 4)

Each user is assigned via independent DV keys:
  DV "ubp_hp_horizontal_v2" → "treatment_fw"      (horizontal assignment)
  DV "ubp_hp_vertical_v3"   → "intent_model__v3"  (vertical assignment)

A user can be in treatment_fw (horizontal) AND intent_model__v3 (vertical)
simultaneously. The two DVs are completely independent.

Result: N x M valid experiment cells, mutually exclusive per layer.
```

The current code already works this way informally (e.g. `hp_vertical_blending_config` DV
is independent of model selection DVs like `enableFWPredictor`). UBP formalizes this.

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

## Phase 1: Vertical Blending Platform

Starting with vertical because:
- More heuristics to replace (7 stages vs 3 for horizontal)
- Existing `VerticalBlendingConfig` JSON is a working config seam to extend from
- Lower risk: carousel ordering is less sensitive to users than within-carousel ordering
- Immediately unblocks NV SR, LCM, PAD experiments

### Step 1: Define `VerticalComponent` interface

**Goal:** Single interface that wraps all carousel types.

**What to build:**
- `interface VerticalComponent` with `id`, `type`, `score` (mutable), `metadata`, `recordTrace()`
- Adapters for all 9 types: `StoreCarousel`, `ItemCarousel`, `CollectionV2`, `DealCarousel`,
  `StoreCollection`, `ItemCollection`, `MapCarousel`, `ReelsCarousel`, `StoreEntity`
- Foundation: `ScorableEntityStoreCarousel` + `ScorableEntityItemCarousel` already exist
  as prototype — extend these, add 7 more

**Key files:**
- `libraries/common/.../scoring/ScorableEntity*.kt` (extend)
- New: `libraries/common/.../ubp/vertical/VerticalComponent.kt`

---

### Step 2: Define `VerticalProcessor` interface + extract 4 existing steps

**Goal:** 4 hardcoded steps become registered, independently-testable processors.

**Processors to build:**
- `ModelScoringProcessor` — wraps `entityScorer.score()` + Sibyl RPC
- `MultiplierBoostProcessor` — wraps `BlendingUtil.blendBundle()` (calibration + intent + boost weights)
- `DiversityRerankProcessor` — wraps `BlendingUtil.rerankEntitiesWithDiversity()`
- `FixedPinningProcessor` — wraps `BoostingBundle.boosted()` + all post-ranking fixups

**Critical design rule:** `params` are injected from config JSON at call time.
Processors do NOT read their own DV keys internally. This is what makes them reusable.

**Key files:**
- New: `libraries/common/.../ubp/vertical/processors/`
- Existing callers untouched during migration (existing code paths still work)

---

### Step 3: Build `VerticalRankingEngine` with processor registry

**Goal:** Config-driven orchestrator that replaces hardcoded `rank()` method.

**What to build:**
- `class VerticalRankingEngine(processorRegistry: Map<String, VerticalProcessor>)`
- `fun rank(components, config, context)`: loop steps, dispatch to registry
- Auto-trace: `recordTrace()` after every step when `config.emitTrace = true`
- Engine has ZERO business logic — pure dispatch only

**Key files:**
- New: `libraries/common/.../ubp/vertical/VerticalRankingEngine.kt`

---

### Step 4: Build `UnifiedExperimentConfig` + `UbpRuntimeUtil`

**Goal:** One JSON file per experiment. Absorb the 10+ scattered config locations.

**What to build:**
- `UnifiedExperimentConfig` data classes (see `context/northstar.md` for schema)
- `UbpRuntimeUtil.resolveConfig(experimentId, experimentMap)` — reads runtime JSON,
  falls back to "control" on missing key or parse error
- Convention: DV label = `"ubp_{experiment_id}"`, JSON = `"ubp/experiments/{experiment_id}.json"`
- Hot-reloadable via `DeserializedRuntimeCache` (same pattern as existing blending config)

**Migration:** Existing `hp_vertical_blending_config` DV + JSON keep working unchanged.
UBP runs alongside. New experiments use new contract.

**Key files:**
- New: `libraries/common/.../ubp/config/UnifiedExperimentConfig.kt`
- New: `libraries/common/.../ubp/config/UbpRuntimeUtil.kt`

---

### Step 5: Wire into `DefaultHomePagePostProcessor`

**Goal:** Route requests with a `ubp_hp_vertical_*` DV to the new engine.

**What to build:**
- In `reOrderGlobalEntitiesV2()`: check for active UBP experiment DV
- If UBP active: convert carousels to `VerticalComponent`s → `VerticalRankingEngine.rank()`
  → apply `sortOrder` back to domain objects
- If not: existing code path unchanged (zero impact to non-UBP traffic)
- Post-ranking fixups (Stage 7) migrate to declared `FIXED_PINNING` steps in config

**Key files:**
- `DefaultHomePagePostProcessor.kt`
- `HomePagePostProcessorV2.kt`

---

### Step 6: Standardize tracing/observability

**Goal:** Every step auto-traces. Stage-wise score snapshots queryable in Snowflake.

**What to build:**
- Standard trace event schema: `{ component_id, step_id, score_before, score_after, request_id }`
- Engine emits automatically when `config.emitTrace = true`
- Replaces manual `IguazuEventUtil` calls per processor

---

## Phase 2 (After Vertical is Live): Horizontal Blending Platform

Same pattern applied to Layer 3:
- `HorizontalComponent` interface (wraps `DiscoveryStore`, `StoreEntity`, `LiteStoreCollection`)
- `HorizontalProcessor` interface with `carousel_overrides` support per `RankingType`
- `HorizontalRankingEngine`
- Absorbs `modifyLiteStoreCollection()` when-chain

S1/S2/S3 favorites reranking becomes an `ORDER_HISTORY_RERANK` processor — first time it's observable.

---

## Phase 3+: Value Function, Calibration, Ads Integration

- Calibration service (`PIECEWISE`, `ISOTONIC`) for cross-type score comparability
- `vAct` weights (GOV / FIV / ads_revenue / strategic) configurable per experiment
- Ads compete on same scale as organic via calibrated `pAct`
- `pImp` position decay modeling

---

## Key Files Reference

| Layer | Key File | What it does |
|---|---|---|
| Vertical entry | `DefaultHomePagePostProcessor.kt` | `reOrderGlobalEntitiesV2()` — 7-stage pipeline |
| Vertical scoring | `EntityRankerConfiguration.kt` | Calls Sibyl + blending |
| Vertical blending | `VerticalBlending.kt` | Calibration × intent × boost weight multipliers |
| Vertical pinning | `Boosting.kt` | `enforceManualSortOrder` enforcement |
| Pinned order config | `PinnedCarouselUtil.kt` | Reads `pinned_carousel_ranking_order.json` |
| Post-ranking fixups | `NonRankableHomepageOrderingUtil.kt` | NV, PAD, gap rules |
| Blending config | `VerticalBlendingConfig.kt` | Existing blending params (to absorb into UnifiedExperimentConfig) |
| Horizontal entry | `DefaultHomePageStoreRanker.kt` | `modifyLiteStoreCollection()` per carousel |
| Horizontal scoring | `StoreCollectionScorer.kt` | Sibyl RPC + feature construction |
| Experiment manager | `DiscoveryExperimentManager.kt` | DV manifest |
| Pipeline DAG | `DefaultHomePagePipeline.kt` | Job dependency graph |

---

## Progress Tracker

- [ ] Step 1: VerticalComponent interface + adapters for all 9 carousel types
- [ ] Step 2: VerticalProcessor interface + extract 4 processors
- [ ] Step 3: VerticalRankingEngine with processor registry + auto-trace
- [ ] Step 4: UnifiedExperimentConfig data classes + UbpRuntimeUtil
- [ ] Step 5: Wire into DefaultHomePagePostProcessor (behind UBP feature flag)
- [ ] Step 6: Standardized tracing / Iguazu schema
- [ ] Phase 2: HorizontalRankingEngine
- [ ] Phase 3+: Value function, calibration, ads integration
