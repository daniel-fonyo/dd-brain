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

### Shadow Validation (DV-gated, parallel execution)

Shadow validation runs **in parallel** with the existing ranking path — not as an if/else branch.
A DV controls whether shadow mode is active. The shadow path swallows all exceptions so it can
never affect the production result.

```kotlin
// Production path — always runs, always returns result to user
val result = rankAndDedupeContent(...)

// Shadow path — runs in parallel when DV is enabled, never affects user
if (ubpShadowEnabled) {
    try {
        val shadowResult = feedRowRanker.rank(rows, pipeline, context)
        compareSortOrders(result, shadowResult)  // log divergences
    } catch (e: Exception) {
        log.warn("UBP shadow failed — swallowed", e)  // never propagates
    }
}

return result  // always the old-path result
```

1. Users always see the old-path result. The shadow path's output is discarded.
2. Compare sort order outputs. Log divergences.
3. Target: `divergence_count = 0` across sustained traffic before proceeding to rollout.

### Rollout (DV-gated, gradual migration)

Once shadow validation proves `divergence_count = 0`, the shadow DV is replaced with a rollout
DV that gates the new path as the **primary** path via an if/else branch:

```kotlin
if (ubpRolloutEnabled) {
    feedRowRanker.rank(rows, pipeline, context)   // new path — serves result to user
} else {
    rankAndDedupeContent(...)                      // old path — unchanged, byte-for-byte
}
```

The rollout DV is ramped gradually: 1% → 5% → 25% → 50% → 100%. At each stage, monitor latency,
error rates, and ranking quality metrics. The old path is the `else` branch — it compiles and runs
identically. Non-UBP traffic is unaffected.

**Characterization tests with UBP flag OFF must remain green at every stage** — proving the old
path is untouched.

---

## What These Abstractions Unlock

### Immediate value

The abstractions proposed above are independently valuable even before the engine is wired in or any traffic is migrated.

**Clear interfaces and abstraction boundaries.** Today, understanding vertical ranking requires tracing through `DefaultHomePagePostProcessor` → `BaseEntityRankerConfiguration` → `EntityScorer` → `BlendingUtil` → `BoostingBundle` → `Ranking` → `NonRankableHomepageOrderingUtil` — a chain of inline method calls across utility objects with no clear contracts between them. The `FeedRow` interface and `FeedRowRankingStep` interface create explicit boundaries: each step has a defined input, output, and parameter contract. Engineers can reason about one step without understanding the entire pipeline.

**Easier to reason about vertical ranking.** Each stage of the ranking pipeline becomes a named, isolated step with typed parameters. Instead of reading 6+ files to understand what happens to a carousel's score, you read one step class. Instead of tracing DV keys across scattered runtime JSONs, you look at one params object. The ranking pipeline becomes a sequence of well-named steps that read top-to-bottom.

**Simplified new carousel type onboarding.** Today, adding a new carousel type means touching 10+ files across the ranking pipeline because type-checks are scattered everywhere. With the `FeedRow` interface, a new carousel type requires one adapter class. The adapter converts to/from `FeedRow`; the engine and all steps work with it automatically. Product teams bringing a new carousel to the homepage no longer need deep HP knowledge or core team pairing.

**Independent testability.** Each step has its own unit tests with injected parameters. Step changes don't require full pipeline regression testing — characterization tests catch unintended side effects at the integration level, and step-level tests verify the step's own behavior. This is a fundamental improvement over the current situation where no tests cover the ranking pipeline.

**Per-step observability.** The engine emits `{ row_id, step_id, score_before, score_after }` after each step, queryable in Snowflake. No ad hoc logging. Debugging ranking behavior becomes: query the trace, see exactly which step changed what score and by how much.

### Horizontal ranking follows the same pattern

Once the vertical abstractions are proven, horizontal ranking (store/item ordering within each carousel) follows the identical architecture: `RowItem` interface with adapters, `RowItemRankingStep` with step wrappers, `RowItemRanker` engine. The incision point is `DefaultHomePageStoreRanker.rank()` wrapping `modifyLiteStoreCollection()`. Same shadow → rollout migration path.

---

## Future: Unified Blending Platform

The abstractions above are the foundation for the full Unified Blending Platform vision. The clean interfaces make the following possible without rearchitecting:

**Config-driven experiments.** MLEs declare `{ experiment_id, model_name, traffic_pct, params }`. The engine resolves the full pipeline from a canonical `control.json` with MLE overrides applied on top. No code changes per experiment. No 2-week HP engineer involvement for parameter tuning.

**Per-layer traffic management.** Hash-based bucket partitioning per layer replaces the DV waterfall. Experiments are mutually exclusive within a layer, orthogonal across layers. An MLE says "I want 5% traffic on vertical" — the router allocates it. No dummy DVs, no priority lists, no reserve segments.

**Unified value function.** Today, organic stores, NV stores, ads, and merchandising carousels are scored on completely different scales by different systems. Nobody can answer: "Is this NV carousel worth more to the user than this Rx carousel at position 3?" The `FeedRow` interface and step architecture make it possible to introduce a shared value function — `EV = pImp(k) × pAct(c) × vAct(c)` — where every content type's score is expressed on a comparable, calibrated scale. Calibration steps (`PIECEWISE`, `ISOTONIC`) become new step types added to the pipeline config. Value weights (`gov_w`, `fiv_w`, `strategic_w`) become step parameters. No engine changes required.

**Ads/organic fair competition.** An `AdsFeedRow` adapter lets ad carousels implement the same `FeedRow` interface. Ads compete in the same pipeline as organic carousels on a shared scoring scale. This replaces the current architecture where ads are inserted after organic ranking by a separate blender (`HomepageAdsBlender`) with no cross-comparison.

All of this is built on the same three interfaces (`FeedRow`, `FeedRowRankingStep`, `FeedRowRanker`) and the same patterns (Adapter, Strategy, Chain of Responsibility) proposed in this document. The abstractions come first; the platform is built on proven foundations.

---
---

# Appendix: Design Patterns, Best Practices, and References

This appendix documents the engineering patterns, legacy code practices, and industry research that inform the design decisions in this proposal. Each section explains the pattern, why it applies, and where it appears in the proposed system.

---

## A. Design Patterns (refactoring.guru / Gang of Four)

### A.1 Adapter Pattern (Structural)

**What it is.** Convert the interface of a class into another interface that clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.

**Why we use it.** The feed-service ranking pipeline operates on 9+ distinct carousel domain types (`StoreCarousel`, `ItemCarousel`, `DealCarousel`, `StoreCollection`, `CollectionV2`, `ItemCollection`, `MapCarousel`, `ReelsCarousel`, `StoreEntity`). Each has a different class structure. The ranking engine needs to treat them uniformly.

**Where it appears.** Each carousel type gets one adapter class that wraps the domain object behind the `FeedRow` interface. The adapter knows about both sides (domain object and `FeedRow`); the engine only knows about `FeedRow`. After ranking, `applyBackTo()` writes final scores back to the original domain object.

```
StoreCarousel  ──→ StoreCarouselRow (adapter) ──→ FeedRow interface
ItemCarousel   ──→ ItemCarouselRow  (adapter) ──→ FeedRow interface
DealCarousel   ──→ DealCarouselRow  (adapter) ──→ FeedRow interface
...9 types total
```

**Reference:** refactoring.guru/design-patterns/adapter

---

### A.2 Strategy Pattern (Behavioral)

**What it is.** Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from the clients that use it.

**Why we use it.** The ranking pipeline has 4 distinct algorithmic stages (scoring, boosting, diversity, pinning). Today they are hardcoded inline methods. We want each stage to be swappable — a different scoring model, a different diversity algorithm — without changing the engine or other stages.

**Where it appears.** Each `FeedRowRankingStep` implementation is a Strategy. The engine doesn't know or care which algorithm runs — it calls `process()` on whatever step the config specifies.

```
FeedRowRankingStep (interface = Strategy)
    ├── ModelScoringStep      — calls Sibyl ML inference
    ├── MultiplierBoostStep   — applies calibration × intent × boost weights
    ├── DiversityRerankStep   — greedy rerank penalizing category density
    └── FixedPinningStep      — enforces configured position pins
```

**Reference:** refactoring.guru/design-patterns/strategy

---

### A.3 Chain of Responsibility (Behavioral)

**What it is.** Pass a request along a chain of handlers. Each handler decides either to process the request or to pass it to the next handler in the chain.

**Why we use it.** The ranking pipeline is a sequence of steps executed in order, each transforming the same list of `FeedRow` objects. Step order is load-bearing: you can't boost before you have scores, can't diversity-rerank before you've boosted, can't pin before final scores exist.

**Where it appears.** The engine loops through steps from the pipeline config and calls each one sequentially:

```
FeedRow list → [ModelScoringStep] → [MultiplierBoostStep] → [DiversityRerankStep] → [FixedPinningStep] → sorted result
```

The chain is config-driven (step sequence comes from JSON), not hardcoded.

**Reference:** refactoring.guru/design-patterns/chain-of-responsibility

---

### A.4 Facade Pattern (Structural)

**What it is.** Provide a simplified interface to a complex subsystem. Facade hides the complexity of config resolution, step dispatch, and tracing behind one method.

**Why we use it.** The caller (`DefaultHomePagePostProcessor`) should not know about the step registry, config schema, parameter deserialization, or trace emission. It calls one method.

**Where it appears.** `FeedRowRanker.rank(rows, pipeline, context)` is the facade. Internally it resolves config, dispatches to steps, handles unknown step types (skip + warn), and emits trace events. The caller sees none of this.

**Reference:** refactoring.guru/design-patterns/facade

---

### A.5 Factory Method (Creational)

**What it is.** Define an interface for creating an object, but let subclasses or configuration decide which class to instantiate.

**Why we use it.** Given an experiment ID and treatment name, the system must produce the correct `ResolvedPipeline` with the right steps and parameters. This resolution logic is centralized in one place.

**Where it appears.** `UbpRuntimeUtil.resolve(experimentId, treatment)` returns a `ResolvedPipeline`. Unknown treatment falls back to control. `extends: "control"` shallow-merges params on top of the control config.

**Reference:** refactoring.guru/design-patterns/factory-method

---

### A.6 Prototype Pattern (Creational)

**What it is.** Create new objects by copying an existing object (the prototype) and modifying the copy.

**Why we use it.** Most experiment treatments are small variations of the control config — change one model name, adjust one boost weight. Copying the entire control config and overriding specific fields is simpler and less error-prone than specifying every field from scratch.

**Where it appears.** `extends: "control"` in experiment config. The engine copies the control pipeline, then shallow-merges the treatment's overrides on top. Only the differences need to be specified.

**Reference:** refactoring.guru/design-patterns/prototype

---

### A.7 Observer Pattern (Behavioral)

**What it is.** Define a one-to-many dependency so that when one object changes state, all its dependents are notified automatically.

**Why we use it.** Steps should have zero tracing code. The engine observes score changes after each step and emits trace events automatically. Steps don't know they're being traced.

**Where it appears.** After each `step.process()` call, the engine snapshots scores and emits a `UbpFeedRowRankingTrace` event (when `emit_trace = true`). The step implementation is completely unaware of tracing.

**Reference:** refactoring.guru/design-patterns/observer

---

### A.8 Null Object Pattern (Behavioral)

**What it is.** Instead of returning null when an object doesn't exist, return a special "do nothing" object that conforms to the expected interface.

**Why we use it.** When an experiment treatment is unknown or missing, the system should not crash — it should gracefully fall back to control behavior.

**Where it appears.** `UbpRuntimeUtil.resolve()` returns the control config when the treatment key is not found. The engine runs the control pipeline without branching on null.

**Reference:** Fowler, *Patterns of Enterprise Application Architecture* — Special Case pattern

---

### A.9 Proxy Pattern (Structural)

**What it is.** Provide a surrogate or placeholder for another object to control access to it.

**Why we use it.** Shadow validation runs both the old and new ranking paths for the same request, comparing outputs without exposing the new path's results to users.

**Where it appears.** The shadow infrastructure intercepts ranking calls, forks execution to both paths, compares sort orders, and logs divergences. Users only see old-path results.

**Reference:** refactoring.guru/design-patterns/proxy

---

### A.10 Template Method — The Pattern Being Replaced

**What it is.** Define the skeleton of an algorithm in a base class, letting subclasses override specific steps without changing the algorithm's structure.

**Why it's relevant.** `BaseEntityRankerConfiguration.rank()` is a Template Method today: score → blend → boost → sort. Subclasses can override individual steps but cannot reorder or skip them. This rigidity is one of the root causes of the current problems.

**What replaces it.** UBP's config-driven Chain of Responsibility. The step sequence comes from JSON, not inheritance. Steps can be added, removed, or reordered via config — no code change required.

**Reference:** refactoring.guru/design-patterns/template-method

---

## B. Legacy Code Practices (Michael Feathers)

All practices in this section are drawn from Michael Feathers' *Working Effectively with Legacy Code* (Prentice Hall, 2004). This is the foundational text on safely modifying code that lacks tests.

### B.1 Cover and Modify (vs. Edit and Pray)

Feathers identifies two approaches to modifying code:

**Edit and Pray:** Understand the code, make changes, poke around to see if it broke. This is how feed-service ranking changes work today. There are no tests covering the ranking pipeline's end-to-end behavior.

**Cover and Modify:** Write tests that lock down current behavior first. Make changes. If tests pass, the refactoring preserved behavior. If they fail, you changed something — investigate or revert.

We use Cover and Modify for every change.

---

### B.2 Characterization Tests

From Feathers: "A characterization test is a test that characterizes the actual behavior of a piece of code."

A characterization test does NOT test what the code should do. It tests what the code actually does right now. The process:

1. Pick the code you want to test.
2. Write a test with a deliberately wrong assertion (e.g., `assertEquals(0, result)`).
3. Run the test — it fails and shows the actual value.
4. Take that actual value and make it the expected value.
5. Now you have a test documenting current behavior.

This sounds backwards. That's the point. You don't know what the code does, so you ask it.

Characterization tests are a temporary safety net. They capture bugs as-is (because users depend on this behavior). They are replaced with proper unit tests after the refactoring is complete.

---

### B.3 Seams and Enabling Points

A seam is a place where you can alter behavior without editing the code at that point. The enabling point is where you decide which behavior to use.

In feed-service, the primary seams are:

**Object seams (constructor injection).** Feed-service uses Spring `@Component` with constructor injection. Every constructor parameter is a seam. In tests, inject mocks or test doubles via constructor params.

**Link seams (Spring configuration).** `@TestConfiguration` and `@Profile("test")` allow swapping entire Spring beans for test doubles.

**Problem areas.** Some ranking logic uses Kotlin `object` singletons (`BlendingUtil`, `PinnedCarouselUtil`). These are NOT seams — you can't swap them via injection. Mitigation: test at a higher level where the object is called, or use MockK's `mockkObject()` as a last resort.

---

### B.4 The Strangler Fig Pattern

Build new functionality alongside old, gradually replace without killing the old system. Both paths coexist until the new path is proven at 100% traffic.

In our system:

```
BEFORE                              AFTER

Request → PostProcessor             Request → PostProcessor
  └─ rankAndMergeContent()            ├─ if (ubpEnabled):
       ├─ Sibyl scoring               │     FeedRowRanker.rank()
       ├─ BlendingUtil                 │       ├─ ModelScoringStep
       └─ Boosting                     │       ├─ MultiplierBoostStep
                                       │       ├─ DiversityRerankStep
                                       │       └─ FixedPinningStep
                                       └─ else:
                                             rankAndMergeContent()
                                             (old path, unchanged)
```

The old path is never deleted until the new path is proven at 100% traffic.

**Reference:** Martin Fowler, "StranglerFigApplication" (2004)

---

### B.5 The Order of Operations

Feathers prescribes a specific order for safely modifying legacy code:

1. **Scratch refactoring** — understand the code (revert when done).
2. **Identify seams** — find where you can alter behavior without editing at that point.
3. **Break dependencies** — minimal, careful changes to make the code testable (extract interface, add constructor parameter).
4. **Write characterization tests** — lock down current behavior at integration and method levels.
5. **Extract abstractions** — the actual work. Characterization tests must stay green after every extraction.
6. **Shadow validate** — run both paths, compare outputs, log divergences.
7. **Switch and clean up** — ramp traffic, replace characterization tests with proper unit tests, delete old code.

---

## C. Clean Code and OOP Principles

### C.1 Composition Over Inheritance (Is-a vs Has-a)

The proposed system uses interfaces and composition, not class hierarchies:

- `FeedRow` IS A rankable unit → interface, not abstract class.
- `StoreCarouselRow` IS A `FeedRow` AND HAS A `StoreCarousel` → implements interface, wraps domain object by composition.
- `FeedRowRankingStep` IS A strategy → interface.
- `ModelScoringStep` IS A `FeedRowRankingStep` AND HAS A `EntityScorer` → dependency injection, not inheritance.

We never extend domain objects (`StoreCarousel`, `ItemCarousel`). They serve a different concern.

**Reference:** Gamma et al., *Design Patterns* (1994) — "Favor object composition over class inheritance."

---

### C.2 Statelessness

- Steps are stateless — all mutable state is on the `FeedRow` objects passed in.
- Engine is stateless — config resolved fresh per call from hot-reloadable cache.
- Adapters are stateless — `toFeedRow()` creates a new wrapper each time.

No shared mutable state between requests. This simplifies testing, avoids concurrency bugs, and makes the system safe for parallel execution.

---

### C.3 Abstraction Boundaries

Clear lines between layers of the system:

- Steps never know about `StoreCarousel`, `ItemCarousel`, etc. — only `FeedRow`.
- Steps never read DV keys — all config flows through injected `params`.
- Engine has zero business logic — only loops, dispatches, traces.
- Adapters are the only code that knows about both domain objects and `FeedRow`.

**The params constraint is absolute.** A step that reads its own DV keys internally is not a UBP step — it defeats the purpose. All tunable behavior flows through `params`.

---

### C.4 Naming Conventions

- Interface names are nouns describing what the thing IS: `FeedRow`, `RowItem`, `FeedRowRankingStep`.
- Implementation names describe what the thing DOES: `ModelScoringStep`, `DiversityRerankStep`.
- No `I` prefix on interfaces (Kotlin convention).
- No `Impl` suffix on implementations — use descriptive names.

---

## D. Summary of References

**Books:**
- Feathers, Michael. *Working Effectively with Legacy Code.* Prentice Hall, 2004.
- Gamma, Helm, Johnson, Vlissides. *Design Patterns: Elements of Reusable Object-Oriented Software.* Addison-Wesley, 1994.
- Fowler, Martin. *Patterns of Enterprise Application Architecture.* Addison-Wesley, 2002.
- Fowler, Martin. *Refactoring: Improving the Design of Existing Code.* Addison-Wesley, 2018 (2nd ed).

**Web references:**
- refactoring.guru/design-patterns (Adapter, Strategy, Chain of Responsibility, Facade, Factory Method, Prototype, Observer, Proxy, Template Method)
- Fowler, Martin. "StranglerFigApplication." martinfowler.com, 2004.
