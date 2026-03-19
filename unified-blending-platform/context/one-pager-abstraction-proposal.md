# Proposal: Ranking Abstraction Layer for Homepage Post-Processing

> 1-pager for stakeholder alignment. Covers the problem, proposed change, and safe delivery plan.

---

## The Problem

Homepage vertical ranking lives in `DefaultHomePagePostProcessor.reOrderGlobalEntitiesV2()`.
The call chain looks like this:

```
reOrderGlobalEntitiesV2()
  ├─ rankAndDedupeContent()
  │   ├─ rankAndMergeContent()
  │   │   ├─ rankContent()
  │   │   │   └─ BaseEntityRankerConfiguration.rank()
  │   │   │       ├─ getEntities()              — flatten carousel types into ScorableEntity list
  │   │   │       ├─ getScoreBundle()            — Sibyl ML scoring (EntityScorer → SibylRegressor)
  │   │   │       ├─ getBoostBundle()            — position boosting, deal multiplier, NV unpin
  │   │   │       ├─ getRankingBundle()          — pin vs flow separation, interleave by score
  │   │   │       └─ getRankableContent()        — re-assemble into typed containers
  │   │   └─ splitPinnedAndRankCarousels()
  │   ├─ PlacementProcessingUtil.rankAdsCandidates()
  │   └─ PlacementProcessingUtil.rankAndChoosePlacementsWithOrganic()
  └─ NonRankableHomepageOrderingUtil.rankWithImmersivesV2()
```

**What's wrong:**

| Pain | Impact |
|---|---|
| **No abstraction layer** | 9+ carousel types handled via type-checks scattered across files. Adding a new carousel type means touching 10+ files. (~1 new type per quarter.) |
| **Hardcoded ranking logic** | Scoring, boosting, blending, pinning, diversity are inline methods in utility objects (`BlendingUtil`, `BoostingBundle`, `Ranking`). Cannot be tested, swapped, or configured independently. |
| **Config fragmentation** | A single ranking experiment's parameters are spread across 6+ locations: DVs, runtime JSONs, hardcoded constants, and Spring beans. No one file describes what an experiment does. |
| **No observability** | No per-step score tracing. Debugging requires reading code and adding ad hoc logging. |
| **Regression fear** | No characterization tests. Changes are "edit and pray" — modify code, manually check if it broke. |

Every stakeholder confirmed this independently (see `context/rfc-feedback.md`).

---

## Proposed Change

Introduce two interfaces and an engine that wrap the existing code — **without rewriting it**.

### 1. `FeedRow` Interface + 9 Adapters

A single interface wrapping all carousel types behind `id`, `type`, `score`, `metadata`, `applyBackTo()`. One adapter per type. Existing code continues to work; adapters are a pure addition.

### 2. `FeedRowRankingStep` Interface + 4 Step Extractions

Each inline ranking method becomes a registered step:

| Step | Wraps | Current Location |
|---|---|---|
| `ModelScoringStep` | Sibyl ML scoring | `EntityScorer.score()` via `BaseEntityRankerConfiguration.getScoreBundle()` |
| `MultiplierBoostStep` | Calibration x intent x vertical boost | `BlendingUtil.blendBundle()` |
| `DiversityRerankStep` | Rx vs non-Rx diversity balancing | `BlendingUtil.rerankEntitiesWithDiversity()` |
| `FixedPinningStep` | Position pinning + boost-by-position | `BoostingBundle.boosted()` → `RankingBundle.ranked()` |

Each step's `process()` method literally calls the same existing method. Steps are wrappers, not rewrites. Parameters are injected from config — steps do not read DVs internally.

### 3. `FeedRowRanker` Engine

Config-driven loop: for each step in the pipeline config, look up the step in a registry, call `process()`. Zero business logic. When wired in, the PostProcessor gets an `if/else` branch:

```
if (ubpEnabled) → FeedRowRanker.rank()     // new path
else            → rankAndDedupeContent()     // old path, byte-for-byte unchanged
```

### 4. Tracing

Auto-trace after each step: `{ row_id, step_id, score_before, score_after, request_id }`. Queryable in Snowflake. Opt-in via config flag.

### 5. Shadow Validation

Run both paths for the same request. Compare sort order. Log divergences. Users only see old-path results. Target: `divergence_count = 0` before any real traffic.

---

## How We Ship This Safely (No Regressions)

Based on *Working Effectively with Legacy Code* (Feathers) — Cover and Modify, never Edit and Pray.

```
Phase 1: CHARACTERIZATION TESTS
  │  Write tests that lock down current behavior of the ranking pipeline.
  │  Integration-level: full pipeline input → golden master output.
  │  Method-level: BlendingUtil, BoostingBundle, EntityScorer individually.
  │  Tests capture what the code DOES, not what it SHOULD do.
  │
Phase 2: EXTRACT ABSTRACTIONS
  │  FeedRow adapters (pure addition, zero risk).
  │  RankingStep wrappers — each step calls the same existing method.
  │  Characterization tests must stay GREEN after every extraction.
  │  Red → stop, investigate, fix or revert.
  │
Phase 3: ENGINE + WIRING
  │  Ship engine (pure addition).
  │  Wire with if/else branch — old path is the else, unchanged.
  │  Characterization tests with UBP flag OFF → must be green.
  │
Phase 4: SHADOW VALIDATE
  │  Both paths, same request. Compare outputs. divergence_count = 0.
  │
Phase 5: RAMP
     1% → 5% → 25% → 50% → 100%. Monitor latency, errors, ranking quality.
     Replace characterization tests with proper unit tests. Delete old path.
```

---

## What This Unlocks

| Before | After |
|---|---|
| New carousel type = touch 10+ files | New carousel type = 1 `FeedRowAdapter` class |
| Ranking experiment = modify inline code across 6 config locations | Ranking experiment = 1 config declaration with step params |
| Debug ranking = read code and add print statements | Debug ranking = query per-step score trace in Snowflake |
| "Does this change break ranking?" = pray | "Does this change break ranking?" = characterization tests + shadow validation |
| Coupled scoring/boosting/blending | Independent steps, independently testable and swappable |

---

## Scope

**In scope:** Vertical ranking pipeline (`DefaultHomePagePostProcessor`). FeedRow, RankingStep, engine, tracing, shadow validation.

**Out of scope (for now):** Horizontal ranking (`DefaultHomePageStoreRanker`), per-layer traffic router, MLE self-service experiment declaration. These follow naturally once vertical abstractions are proven.

---

## Timeline Dependency

Steps are sequenced:

```
[FeedRow interface + adapters] ──→ [RankingStep extraction] ──→ [Engine] ──→ [Wiring] ──→ [Tracing] ──→ [Shadow]
         (pure addition)              (wrap existing methods)    (loop)     (if/else)     (observe)     (prove)
```

Each step is independently mergeable and delivers incremental value.
