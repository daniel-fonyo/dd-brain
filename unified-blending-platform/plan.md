# Unified Blending Platform — Plan

## Status: RFC in progress — POC on private branch validates design

> **Source of truth**: `Unified Blending Platform Vision.md` (the RFC), `Unified Blending Platform 1 Pager.md`, `poc-generic-ranking.md` (engine design).
>
> **Pivot (March 2026):** After stakeholder conversations (YZ, Dipali, Frank), approach
> is "ship good abstractions in feed-service, extract seams, build the engine on proven
> foundations." Per-layer traffic splitting is a **future design** — we'll tackle it after
> interfaces and abstractions are shipped and proven.
>
> See `context/pivot-analysis.md` and `context/rfc-feedback.md` for full synthesis.

---

## Approach: Working Effectively with Legacy Code

The feed-service post-processing code is messy and poorly understood (confirmed by every
stakeholder). Rewriting is too risky. Instead:

1. **Extract interfaces at seams** — `Rankable` formalizes what 9+ types already have (`id`, `predictionScore`). `RankingStep` wraps hardcoded ranking methods. Old code still works.
2. **Ship abstractions first** — Each abstraction is independently useful. `Rankable` solves Frank's new-carousel-type problem. `RankingStep` makes ranking composable.
3. **Build the engine on proven foundations** — Once `Rankable` and `RankingStep` exist and are tested, the `RankingPipeline` is just a chain of handlers. Ship when abstractions are proven.
4. **Traffic splitting later** — Per-layer traffic management is a separate design effort after the engine is wired in. See `experiment-traffic-industry-research.md` for the design space.

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
| **Frank Zhang** | NV BE | New carousel type onboarding — no abstraction | `Rankable` interface |
| **Yu Zhang** | Manager | DV waterfall — experiment traffic management | Per-layer traffic router (future) |
| **Dipali Ranjan** | MLE | Model experiment lifecycle — config mess | Simplified experiment declaration |

All agree: code is messy, no abstraction layer, adding anything requires deep HP knowledge.

---

## Naming

| Concept | Name | Description |
|---|---|---|
| Unified interface | `Rankable` | Any type with `rankableId()` + `predictionScore` + `withPredictionScore()`. Not a wrapper — existing domain types implement it directly. |
| Step | `RankingStep<S : Enum<S>>` | Domain logic contract. Items in → items out. Parameterized by step type enum. |
| Handler | `RankingHandler` | `fun interface` — infrastructure wrapper (metrics, conditions, shadow). Chain of Responsibility pattern. |
| Engine | `RankingPipeline<S : Enum<S>>` | Assembles handler chain from step registry, executes via `foldRight`. Zero business logic. |
| Carousel rank step enum | `CarouselRankStepType` | Inter-carousel ranking. Phase 1: `RANK_ALL`. Phase 2: `MODEL_SCORING`, `MULTIPLIER_BOOST`, `DIVERSITY_RERANK`, `POSITION_BOOSTING`, `FIXED_PINNING`. |
| Intra-carousel rank step enum | `IntraCarouselRankStepType` | Within-carousel store ranking. Phase 1: `RANK_ALL`. Phase 2: granular steps TBD. |

**Why not "Vertical/Horizontal"?** "Vertical" is overloaded with business verticals (grocery, convenience landing pages). `CarouselRank` / `IntraCarouselRank` removes ambiguity.

**Key design choice**: `Rankable` uses interface inheritance (existing types add `override` + one-line `withPredictionScore`), NOT composition wrappers. The fields already exist — we're formalizing a convention into a compile-time contract.

> **POC branch**: `feed-service: feat/vertical-ranking-abstraction-phase1` — uses old naming (`VerticalStepType`, `VerticalRankAllStep`). This is a private POC to validate feasibility, not production code. Will be rewritten post-RFC approval with correct naming.

---

## Internal Engine Architecture

Phase 1: **one `RANK_ALL` step** wrapping the entire legacy pipeline. This proves the interfaces work end-to-end without changing any ranking behavior.

Phase 2: **decompose into granular steps** once `RANK_ALL` is stable. The engine is unchanged — add enum values + step implementations + new pipeline config.

```
Phase 1:  RANK_ALL
Phase 2:  MODEL_SCORING → MULTIPLIER_BOOST → DIVERSITY_RERANK → POSITION_BOOSTING → FIXED_PINNING
```

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

## Implementation Steps

### Step 1: `Scorable` interface on existing domain types

**Goal:** Single interface formalizing what 12 types already have. Immediately useful — new carousel types only need to implement `Scorable` + `SortablePlacement`. Solves Frank's #1 pain.

**What to build:**
- `interface Scorable { val id: String; val predictionScore: Double?; fun withPredictionScore(score: Double): Scorable }`
- Add `Scorable` to all 9 vertical types: `StoreCarousel`, `ItemCarousel`, `DealCarousel`, `StoreCollection`, `CollectionV2`, `ItemCollection`, `MapCarousel`, `ReelsCarousel`, `StoreEntity`
- Add `Scorable` to 3 horizontal types: `StoreEntity`, `ItemStoreEntity`, `DealStoreEntity`
- Each type: add `override` to existing fields + one-line `withPredictionScore` via `copy()`

**Key files:**
- New: `libraries/common/.../ubp/Scorable.kt`
- Modified: 12 existing data classes (add `: Scorable` + `override` + `withPredictionScore`)

**Open question:** `LiteStoreCollection` has no `predictionScore` field (uses maps). Add field? Exclude?
**Open question:** `StoreEntity` has `id: Long`. Scorable expects `String`. Use `toString()`?

**Why first:** Pure seam extraction. Zero behavior change. Compile-time enforcement of existing convention.

---

### Step 2: `RankingStep` + `RankingHandler` + `Ranker` engine

**Goal:** Extract legacy ranking into one step, wrap in handler chain, orchestrate via Ranker.

**What to build:**
- `interface RankingStep<S : Enum<S>>` — domain logic contract
- `interface RankingHandler` — chain of responsibility (metrics, conditions)
- `class Ranker<S : Enum<S>>` — assembles chain from pipeline config, executes
- `VerticalRankAllStep` — wraps entire `BaseEntityRankerConfiguration.rank()`
- `HorizontalRankAllStep` — wraps entire `StoreCarouselService.sortEntities()`

**Key files:**
- New: `libraries/common/.../ubp/RankingStep.kt`
- New: `libraries/common/.../ubp/RankingHandler.kt` (BaseHandler, StepHandler, MetricsHandler)
- New: `libraries/common/.../ubp/Ranker.kt`
- New: `libraries/common/.../ubp/vertical/VerticalRankAllStep.kt`
- New: `libraries/common/.../ubp/horizontal/HorizontalRankAllStep.kt`

**Why second:** Consumes `Scorable` (depends on Step 1). Wraps legacy — no behavior change.

---

### Step 3: Wire engine into feed-service

**Goal:** Route requests through `Ranker<VerticalStepType>` using `RANK_ALL`. Zero impact — same behavior, new path.

**Vertical incision:** inside `reOrderGlobalEntitiesV2()`, before `rankAndDedupeContent()`:
```kotlin
val items = carousels.filterIsInstance<Scorable>()
val ranked = verticalRanker.rank(items, listOf(VerticalStepType.RANK_ALL), ctx)
```

**Horizontal incision:** inside `DefaultHomePageStoreRanker.rank()`, wrapping `modifyLiteStoreCollection()`.

**Key files:**
- `pipelines/homepage/.../DefaultHomePagePostProcessor.kt`
- `pipelines/homepage/.../DefaultHomePageStoreRanker.kt`

---

### Step 4: Shadow validation ✓

**Goal:** Prove `RankingPipeline` + `RANK_ALL` produces identical output to old path.

- Shadow runs parallel with legacy via `coroutineScope { async(shadowCoroutineContext) {} }`
- 5s `withTimeoutOrNull` safeguard — shadow never blocks requests
- Shadow result always discarded — users see legacy result
- `emitShadowComparisonMetrics()` compares carousel ID ordering

**Sandbox results (2026-03-23, sandbox `rd3f1fb7`):**
- 450 `[UBP-SHADOW]` log entries across multiple consumers
- 90 `orderMatch=true`, 1 `orderMatch=false` (99% match)
- 0 timeouts, 0 failures
- Shadow runs on `shadow-coroutine-worker` threads, legacy on `app-coroutine-worker` — confirmed true parallelism
- Typical timing: shadow 115ms, legacy 135ms, total 135ms (= max, not sum)
- The 1 mismatch needs investigation (likely non-deterministic scoring tie-breaking)

---

### Step 5: Standardized tracing

**Goal:** Auto-trace per step. Score snapshots queryable in Snowflake.

- Standard event: `{ item_id, step_type, score_before, score_after, request_id }`
- `MetricsHandler` already captures duration; add score diffing
- New proto: `ubp_ranking_trace.proto` in services-protobuf

---

### Step 6: Granular step decomposition (Phase 2)

**Goal:** Break `RANK_ALL` into granular steps. Engine unchanged — add enum values + implementations.

```kotlin
enum class VerticalStepType {
    RANK_ALL, MODEL_SCORING, MULTIPLIER_BOOST, DIVERSITY_RERANK, POSITION_BOOSTING, FIXED_PINNING,
}
```

Each new step wraps the same existing methods, but independently testable and configurable.

---

### Future: Per-layer traffic splitting

After interfaces and engine are shipped and proven, design the traffic management system:
- Hash-based bucket partitioning per layer (replaces DV waterfall)
- MLE declares experiment_id + layer + model + traffic_pct
- UBP assigns users via `murmurHash(consumerId, layerSalt) % 1000`
- See `context/experiment-traffic-industry-research.md` for design space

This solves YZ's #1 pain (DV waterfall). Depends on working engine with config-driven experiments.

---

## N x M Experiment Model (Target State)

Experiments mutually exclusive WITHIN a layer, orthogonal ACROSS layers.

```
Layer: vertical    → Experiment A (5%), Experiment B (10%), Control (85%)
Layer: horizontal  → Experiment C (3%), Control (97%)
→ Any (vertical, horizontal) combination is valid
```

No DV waterfall. No dummy DVs. No reserve segments. No priority lists.

---

## New Carousel Type Onboarding (Frank's Pain)

Once `Rankable` is accepted, onboarding a new carousel type:

```
1. Add `: Rankable` to the new data class (1 line per field + withPredictionScore)
2. Stage 1 — Pin: hardcode sort order (existing pattern, no new code)
3. Stage 2 — Explore: enable UCB exploration for the new type (config change once engine exists)
4. Stage 3 — Organic: let model rank naturally
```

Before UBP: touch 10+ files, need HP team pairing, 2-3 weeks.
After UBP: implement `Rankable` on one class, done.

---

## Key Files Reference

| Layer | Key File | What it does |
|---|---|---|
| Vertical entry | `DefaultHomePagePostProcessor.kt` | `reOrderGlobalEntitiesV2()` — incision point |
| Vertical scoring | `EntityRankerConfiguration.kt` | Calls Sibyl + blending (wrapped by `VerticalRankAllStep`) |
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

## UBP Naming History

Resolved naming collisions with existing monorepo types:

| Current name | POC branch name | Collision resolved |
|---|---|---|
| `Rankable` | `Scorable` | sdk-core's `Scorable` (ML feature extraction interface) |
| `RankingPipeline` | `Ranker` | sdk-p13n's `abstract class Ranker` |
| `CarouselRankStepType` | `VerticalStepType` | "vertical" overloaded with business verticals |
| `IntraCarouselRankStepType` | `HorizontalStepType` | clarity over brevity |

See `CLAUDE.md` naming conventions table for the full canonical list.

---

## Progress Tracker

**RFC**
- [ ] Finalize RFC (`spec/rfc-abstraction-layer.md`) — ready for review
- [ ] Copy to Google Docs, email `rfc@doordash.com`
- [ ] Get sign-off from Yu, Frank, Dipali

**POC (private branch — validates design, not production code)**
- [x] POC: `Rankable` interface on 9 carousel domain types
- [x] POC: `RankingStep` + `RankingHandler` + `RankingPipeline` engine + `RANK_ALL` step
- [x] POC: Wire engine into `DefaultHomePagePostProcessor.rankContent()` (shadow mode)
- [x] POC: Shadow validation on sandbox (450 log entries, 99% orderMatch, 0 failures, parallel coroutines confirmed)
- Branch: `feed-service: feat/vertical-ranking-abstraction-phase1` (uses old naming — will be rewritten post-RFC)

**Phase 1: Carousel rank interfaces (post-RFC approval)**
- [ ] Implement with correct naming (`CarouselRankStepType`, `CarouselRankAllStep`)
- [ ] Shadow validation on production traffic
- [ ] Intra-carousel ranking (`IntraCarouselRankStepType`, `IntraCarouselRankAllStep`)
- [ ] Standardized tracing

**Phase 2: Granular Steps (decompose RANK_ALL)**
- [ ] Step 6: Break vertical `RANK_ALL` into `MODEL_SCORING`, `MULTIPLIER_BOOST`, `DIVERSITY_RERANK`, `POSITION_BOOSTING`, `FIXED_PINNING`
- [ ] Break horizontal `RANK_ALL` into granular steps

**Future: Traffic + Experiments**
- [ ] Per-layer traffic splitting design + implementation
- [ ] MLE self-service experiment declaration
- [ ] N × M cross-layer experiment support

**Future: Advanced**
- [ ] `CalibrationStep` (`PIECEWISE`, `ISOTONIC`)
- [ ] `ValueFunctionStep` (pImp x pAct x vAct)
- [ ] Ads/organic fair competition via unified Rankable
