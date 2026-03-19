# UBP Implementation Plan — Overview

## Guiding Principles

### Design Patterns (refactoring.guru)
Every structural decision references a named pattern. This is not decoration — patterns give
engineers shared vocabulary for review and onboarding.

| Pattern | Where used in UBP |
|---|---|
| **Adapter** (Structural) | `FeedRow` adapters wrap 9 typed carousel domain objects |
| **Strategy** (Behavioral) | Each `FeedRowRankingStep` is a swappable algorithm |
| **Chain of Responsibility** (Behavioral) | Steps execute in sequence, each passing the list to the next |
| **Facade** (Structural) | `FeedRowRanker.rank()` hides registry, config resolution, and tracing |
| **Factory Method** (Creational) | `UbpRuntimeUtil.resolveConfig()` produces the right config for an experiment |
| **Prototype** (Creational) | `extends: "control"` shallow-merges a copy of control config |
| **Proxy** (Structural) | Shadow infrastructure intercepts ranking calls for comparison |
| **Observer** (Behavioral) | Engine auto-traces after each step without step code knowing |
| **Null Object** (Behavioral) | Missing experiment key returns control config, not null |

### Clean Code / Pragmatic Programming Choices

**Is-a vs Has-a:**
- `FeedRow` IS A rankable unit → interface, not abstract class
- `StoreCarouselRow` IS A `FeedRow` AND HAS A `StoreCarousel` → implements interface, wraps domain object by composition
- `FeedRowRankingStep` IS A strategy → interface
- `ModelScoringStep` IS A `FeedRowRankingStep` AND HAS A `EntityScorer` → DI, not inheritance
- Never extend domain objects (`StoreCarousel`, `ItemCarousel`) — they serve a different concern

**Naming:**
- Interface names are nouns describing what the thing IS: `FeedRow`, `RowItem`, `FeedRowRankingStep`
- Implementation names describe what the thing DOES: `ModelScoringStep`, `DiversityRerankStep`
- No `I` prefix on interfaces (Kotlin convention)
- No `Impl` suffix on implementations — use descriptive names

**Abstraction boundaries:**
- Steps never know about `StoreCarousel`, `ItemCarousel`, etc. — only `FeedRow`
- Steps never read DV keys — all config flows through `params`
- Engine has zero business logic — only loops, dispatches, traces
- Adapters are the only code that knows about both domain objects and `FeedRow`

**Statelessness:**
- Steps are stateless — all mutable state is on the `FeedRow` objects passed in
- Engine is stateless — config resolved fresh per call from hot-reloadable cache
- Adapters are stateless — `toFeedRow()` creates a new wrapper each time

---

## Implementation Order and Dependencies

Parts must be built in this order. Each part is a prerequisite for the next.

```
Part 1: Shadow Infrastructure
  ↓  (validates equivalence before real traffic)
Part 5: UnifiedExperimentConfig + UbpRuntimeUtil
  ↓  (config must exist before engine can read it)
Part 2: FeedRow Interface + Adapters
  ↓  (interface must exist before steps can operate on it)
Part 3: FeedRowRankingStep Interface + 4 Implementations
  ↓  (steps must exist before engine can dispatch to them)
Part 4: FeedRowRanker Engine + Registry
  ↓  (engine must exist before it can be wired in)
Part 6: Wiring into Pipeline
  ↓  (engine live but behind shadow DV)
Part 7: Tracing
     (can be built in parallel with 3/4, but completes after 6)
```

Part 1 (shadow) is deliberately first. It gives you the comparison harness before any engine
code exists. You build the rest knowing that the shadow will validate correctness continuously.

---

## What Each Part Produces

| Part | Deliverable | Tests |
|---|---|---|
| 1 | Shadow branch in post-processor and store-ranker, Snowflake comparison query | Integration test: shadow produces same output as control |
| 2 | `FeedRow` interface, `RowType` enum, 9 adapter classes | Unit tests per adapter: `toFeedRow()` + `applyBackTo()` roundtrip |
| 3 | `FeedRowRankingStep` interface, 4 step classes | Unit tests per step with injected params, no DV reads |
| 4 | `FeedRowRanker`, `stepRegistry` wiring | Unit test: known steps execute in order; unknown step skipped |
| 5 | `UnifiedExperimentConfig` data classes, `UbpRuntimeUtil` | Unit test: `extends` merge semantics, fallback to control |
| 6 | Branch in `reOrderGlobalEntitiesV2()` and `rank()` | Integration test: no regression on non-UBP traffic |
| 7 | `UbpTraceEmitter`, proto file | Unit test: emit called after each step when `emit_trace=true` |

---

## Horizontal vs Vertical Scope Per Part

Most parts address vertical (Phase 1). Horizontal (Phase 2) reuses the same interfaces with
different implementations. Where a part applies to both phases, this is noted explicitly.

Parts 1, 2, 3, 4, 5, 6, 7 all apply to **vertical Phase 1**.
Parts 2, 3, 4, 5, 6, 7 have direct **horizontal Phase 2** equivalents with the same structure
but different concrete classes (`RowItem` instead of `FeedRow`, `RowItemRankingStep`, etc.).

---

## Phase 1 Scope Boundary — What UBP Does Not Handle

UBP Phase 1 replaces **only** `rankAndMergeContent()` in `reOrderGlobalEntitiesV2()`. Everything
listed below stays unchanged. Do not attempt to absorb these into UBP steps.

### Post-ranking business rule fixups (stay in `NonRankableHomepageOrderingUtil`)

These run AFTER the UBP engine returns. They are business constraints, not ranking experiments.
They are not MLE-configurable and do not belong in experiment config JSON.

| Fixup | Why it stays outside UBP |
|---|---|
| NV carousel pinning (`updateSortOrderOfNVCarousel`) | Context-driven at request time (post-checkout boosting state), not a static experiment param |
| Member pricing priority (`updateSortOrderOfMemberPricing`) | Runtime feature flag reading a hardcoded carousel ID list from `DeserializedRuntimeCache` |
| PAD / Taste of DashPass positioning (`updateSortOrderOfTasteOfDashPass`) | Controlled by its own separate DV (`pad_carousel_spotlight_v2`), independent of UBP config |
| Color bleed reordering | Presentation constraint — adjacent same-brand color prevention, not ranking |
| Immersive content spacing | Presentation constraint — gap rules between immersive content types, not ranking |

**`FIXED_PINNING` step scope**: The `FIXED_PINNING` step in UBP is only for MLE-configured pins
declared in experiment JSON (e.g. "for this diversity experiment, hold NV_CAROUSEL at position 0").
It does NOT replace the hardcoded NV post-checkout pinning or PAD positioning above.

### Ads (stay as post-ranking insertion)

Ads are assembled by a separate code path (`PlacementProcessingUtil.rankAdsCandidates()`,
`maybeAddSponsoredCarousel()`) that runs outside `rankAndMergeContent()`. They are not part of
`HomePageStoreLayoutOutputElements` and therefore not in `toFeedRows()`.

Phase 1 UBP has no `AD_CAROUSEL` FeedRow. Ads compete in a unified list in Phase 3+ only.

### Value function — Phase 1 approximation

```
Full EV  = pImp(k) × pAct(c) × vAct(c)

Phase 1  =    1.0  ×  MODEL_SCORING  ×  MULTIPLIER_BOOST
```

- `pImp(k)` (position decay) is **not implemented in Phase 1** — steps run before final position
  is known
- `MODEL_SCORING` computes `pAct(c)` — the Sibyl CTR/CVR/order probability
- `MULTIPLIER_BOOST` approximates `vAct(c)` — boost weights encode business value differences
  between content types at equal model score
- Step order in config matters: `MODEL_SCORING` must precede `MULTIPLIER_BOOST`
- Phase 3 introduces explicit `pImp` decay and explicit `vAct` weight config
