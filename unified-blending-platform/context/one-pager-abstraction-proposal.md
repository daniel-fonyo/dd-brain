# Proposal: Ranking Abstraction Layer for Homepage Post-Processing

---

## Problem

Homepage vertical ranking lives in `DefaultHomePagePostProcessor.reOrderGlobalEntitiesV2()`.
The ranking pipeline is a chain of inline method calls through utility objects:

```
reOrderGlobalEntitiesV2()
  └─ rankAndDedupeContent()
       └─ rankAndMergeContent()
            └─ rankContent()
                 └─ BaseEntityRankerConfiguration.rank()
                      ├─ getEntities()         — flatten 9+ carousel types into ScorableEntity
                      ├─ getScoreBundle()       — Sibyl ML scoring
                      ├─ getBoostBundle()       — position boosting, deal multiplier, NV unpin
                      ├─ getRankingBundle()     — pin vs flow separation, sort by score
                      └─ getRankableContent()   — re-assemble into typed containers
```

**What's wrong:**

- **No abstraction layer.** 9+ carousel types handled via type-checks scattered across files. Adding a new carousel type means touching 10+ files. (~1 new type per quarter.)
- **Hardcoded ranking logic.** Scoring, boosting, blending, pinning, and diversity are inline methods in utility objects (`BlendingUtil`, `BoostingBundle`, `Ranking`). They cannot be tested, swapped, or configured independently.
- **Config fragmentation.** A single ranking experiment's parameters live in 6+ locations: DVs, runtime JSONs, hardcoded constants, and Spring beans. No single file describes what an experiment does.
- **No observability.** No per-step score tracing. Debugging requires reading code and adding ad hoc logging.
- **No safety net.** No characterization tests cover the ranking pipeline. Changes are "edit and pray."

Every stakeholder confirmed these pains independently (Frank Zhang, Yu Zhang, Dipali Ranjan).

---

## Goals

1. **Add abstraction interfaces** at the seams of the ranking pipeline so ranking logic is composable, testable, and independently configurable.
2. **Preserve all existing behavior** — wrap existing methods, don't rewrite them. The old code path remains unchanged and continues to serve all traffic until proven safe.
3. **Establish a characterization test suite** that locks down current ranking behavior before any abstractions are introduced.

## Non-Goals

- Rewriting ranking logic. Steps call the same existing methods.
- Changing any experiment behavior or traffic allocation.
- Horizontal ranking (`DefaultHomePageStoreRanker`) — follows once vertical is proven.
- MLE self-service experiment declaration — future work built on these abstractions.

---

## Step 1: Characterization Tests (Safety Net)

Before introducing any abstraction, we lock down the current behavior using characterization tests
(from Feathers' *Working Effectively with Legacy Code*).

A characterization test captures what the code **actually does**, not what it **should** do. If we later
change something and a characterization test fails, we know behavior shifted — intentional or not.

**Two levels:**

| Level | What it covers | Example |
|---|---|---|
| **Integration** | Full pipeline: fixed input → golden master output | `reOrderGlobalEntitiesV2()` with fixture request + mocked Sibyl → assert sort order matches golden master |
| **Method** | Individual ranking methods we'll wrap | `BlendingUtil.blendBundle()` with known config → assert score output. `BoostingBundle.boosted()` with known input → assert order. |

**Non-determinism handled by:** mocking Sibyl at client level, injecting `Clock.fixed()`, loading
experiment maps from test fixtures (not live DV), scrubbing UUIDs.

**Rule:** All characterization tests must pass consistently (run 3x for flakiness) before any
abstraction code is written.

---

## Step 2: Proposed Abstractions (Additions at the Seams)

Every change below is a **pure addition**. No existing file's behavior is modified. The existing
ranking path continues to compile and run exactly as it does today.

### `FeedRow` Interface + 9 Adapters

A single interface wrapping all carousel types: `id`, `type`, `score` (mutable), `metadata`, `applyBackTo()`.

One adapter class per carousel type. Each adapter converts to/from `FeedRow` and the existing domain
object (e.g., `ScorableEntityStoreCarousel`). The existing domain objects are unchanged.

```
New files (pure additions):
  ubp/vertical/FeedRow.kt              — interface
  ubp/vertical/RowType.kt              — enum of 9 carousel types
  ubp/vertical/adapters/*.kt           — 9 adapter classes

Existing files modified: NONE
```

### `FeedRowRankingStep` Interface + 4 Step Wrappers

Each inline ranking method gets a step wrapper. The step's `process()` calls the **same existing
method** — same class, same arguments, same logic. The step is a thin delegation layer.

| Step class | Wraps (existing method) | Current location |
|---|---|---|
| `ModelScoringStep` | Sibyl ML scoring | `EntityScorer.score()` via `BaseEntityRankerConfiguration.getScoreBundle()` |
| `MultiplierBoostStep` | Calibration x intent x vertical boost weights | `BlendingUtil.blendBundle()` |
| `DiversityRerankStep` | Rx vs non-Rx diversity balancing | `BlendingUtil.rerankEntitiesWithDiversity()` |
| `FixedPinningStep` | Position pinning + boost-by-position | `BoostingBundle.boosted()` + `RankingBundle.ranked()` |

Parameters are injected from config. Steps do not read DVs or runtime JSON internally.

```
New files (pure additions):
  ubp/vertical/FeedRowRankingStep.kt         — interface
  ubp/vertical/steps/ModelScoringStep.kt
  ubp/vertical/steps/MultiplierBoostStep.kt
  ubp/vertical/steps/DiversityRerankStep.kt
  ubp/vertical/steps/FixedPinningStep.kt

Existing files modified: NONE
```

### `FeedRowRanker` Engine

Config-driven loop: read step sequence from pipeline config, look up each step in a registry,
call `process()`. Zero business logic — pure dispatch.

```
New files (pure additions):
  ubp/vertical/FeedRowRanker.kt        — engine
  ubp/vertical/StepRegistry.kt         — step lookup

Existing files modified: NONE
```

### Wiring (the only existing-file change)

One `if/else` branch added to `DefaultHomePagePostProcessor`:

```kotlin
if (ubpEnabled) {
    feedRowRanker.rank(rows, pipeline, context)   // new path
} else {
    rankAndDedupeContent(...)                      // old path — unchanged, byte-for-byte
}
```

The old path is the `else` branch. It compiles and runs identically. Non-UBP traffic is unaffected.

**Characterization tests with UBP flag OFF must remain green** — proving the old path is untouched.

### Shadow Validation

Before any real traffic uses the new path:

1. Run **both** paths for the same request.
2. Compare sort order outputs.
3. Log divergences.
4. Users only see old-path results.
5. Target: `divergence_count = 0`.

---

## What These Abstractions Unlock (Future)

The abstractions above are independently valuable, but they also create the foundation for larger
improvements that are currently impossible:

### Near-term

| Capability | How abstractions enable it |
|---|---|
| **New carousel type onboarding** | New type = 1 `FeedRowAdapter` class instead of touching 10+ files. Product teams can onboard without core team pairing. |
| **Per-step observability** | Engine emits `{ row_id, step_id, score_before, score_after }` after each step. Queryable in Snowflake. No ad hoc logging. |
| **Independent step testing** | Each step has its own unit tests. Step changes don't require full pipeline regression testing — characterization tests catch unintended side effects. |

### Medium-term

| Capability | How abstractions enable it |
|---|---|
| **Config-driven experiments** | MLEs declare `{ experiment_id, model_name, traffic_pct, params }`. UBP resolves the full pipeline. No code changes per experiment. |
| **Per-layer traffic management** | Hash-based bucket partitioning per layer (`vertical`, `horizontal`). Experiments are mutually exclusive within a layer, orthogonal across layers. Eliminates the DV waterfall. |
| **Horizontal ranking** | Mirror the same pattern: `RowItem` interface, `RowItemRankingStep`, `RowItemRanker`. Plug into `DefaultHomePageStoreRanker` with the same if/else wiring. |

### Long-term

| Capability | How abstractions enable it |
|---|---|
| **Calibration step** | New `CalibrationType.PIECEWISE` or `ISOTONIC` step — add to pipeline config, no engine changes. |
| **Value function step** | `pImp x pAct x vAct` composite scoring — one new step class, one config entry. |
| **Ads/organic fair competition** | `AdsFeedRow` adapter lets ads compete in the same pipeline as organic carousels. |
