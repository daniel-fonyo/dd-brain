# RFC Diagrams — Text References for Visual Design

> These are text-character reference diagrams for the Ranking Abstraction Layer RFC.
> Use these as blueprints when building visuals in diagram software.
> All diagrams reflect the **shipped Phase 1 implementation** (Scorable + Ranker<S> + RANK_ALL).

---

## Diagram 1: The Homepage Grid (Mental Model)

```
                        ◄─────── horizontal ranking ───────►

                   slot 0       slot 1       slot 2       slot 3
                ┌────────────────────────────────────────────────┐
    row 0       │  [Store A]   [Store B]   [Store C]   [Store D] │  Favorites
                ├────────────────────────────────────────────────┤
    row 1       │  [Store E]   [Store F]   [Store G]   [Store H] │  Taste
  ▲             ├────────────────────────────────────────────────┤
  │ vertical    │  [Item X]    [Item Y]    [Item Z]              │  Items
  │ ranking     ├────────────────────────────────────────────────┤
  ▼             │  [Store I]   [Store J]                         │  NV
                └────────────────────────────────────────────────┘

   Two independent ranking problems:
   ─────────────────────────────────
   VERTICAL  =  which carousel goes at which row?
   HORIZONTAL =  within a row, which store/item goes at which slot?
```

---

## Diagram 2: The Problem — No Shared Interface (Before)

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    CURRENT STATE: 9 TYPES × 4 STAGES            │
  │                    = 36 type-check branches                     │
  └─────────────────────────────────────────────────────────────────┘

  StoreCarousel ──┐
  ItemCarousel  ──┤
  DealCarousel  ──┤
  StoreCollection─┤    SCORE STAGE                    BLEND STAGE
  CollectionV2  ──┼──► if StoreCarousel? ...  ──────► if StoreCarousel? ...
  ItemCollection──┤    if ItemCarousel?  ...          if ItemCarousel?  ...
  MapCarousel   ──┤    if DealCarousel?  ...          if DealCarousel?  ...
  ReelsCarousel ──┤    if StoreCollection? ...        if StoreCollection? ...
  StoreEntity   ──┘    ... (9 branches)               ... (9 branches)
                              │                              │
                              ▼                              ▼
                       BOOST STAGE                     PIN STAGE
                       if StoreCarousel? ...           if StoreCarousel? ...
                       if ItemCarousel?  ...           if ItemCarousel?  ...
                       ... (9 branches)                ... (9 branches)

  ─────────────────────────────────────────────────────────────────────
  Adding 1 new carousel type = touch 10+ files, modify 36+ branches
  Changing 1 experiment param = 10-15 files, 2-3 weeks of HP eng time
  Test coverage on ranking pipeline = ZERO
  ─────────────────────────────────────────────────────────────────────
```

---

## Diagram 3: The Core Idea — Interface Inheritance, Not Wrappers

```
  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          ┌─────────────┐
  │ StoreCarousel│ │ ItemCarousel│ │ DealCarousel│   ...    │ StoreEntity │
  │  : Scorable  │ │  : Scorable │ │  : Scorable │          │  : Scorable │
  │             │ │             │ │             │          │             │
  │ id          │ │ id          │ │ id          │          │ id          │
  │ prediction  │ │ prediction  │ │ prediction  │          │ prediction  │
  │   Score     │ │   Score     │ │   Score     │          │   Score     │
  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘          └──────┬──────┘
         │               │               │                        │
         │   Fields already exist on these types.                 │
         │   Scorable formalizes the convention                   │
         │   into a compile-time contract.                        │
         │               │               │                        │
         ▼               ▼               ▼                        ▼
  ╔═══════════════════════════════════════════════════════════════════╗
  ║                     toScorableList()                              ║
  ║    RankableContent → List<Scorable>                              ║
  ║    Flattens all carousel fields into one typed list              ║
  ╚═══════════════════════════════════════════════════════════════════╝
                              │
                              ▼
  ╔═══════════════════════════════════════════════════════════════════╗
  ║                     RANKING ENGINE                               ║
  ║            Ranker<VerticalStepType>                               ║
  ║                                                                  ║
  ║   Builds handler chain from step types                           ║
  ║   Executes: step.execute(items, context)                         ║
  ║                                                                  ║
  ║   Zero type-checks. Zero business logic. Pure dispatch.          ║
  ╚═══════════════════════════════════════════════════════════════════╝
                              │
                              ▼
  ╔═══════════════════════════════════════════════════════════════════╗
  ║                     toRankableContent()                          ║
  ║    List<Scorable> → RankableContent                              ║
  ║    Filters instances back into typed container fields            ║
  ║    Round-trip preserves all items                                ║
  ╚═══════════════════════════════════════════════════════════════════╝

  ─────────────────────────────────────────────────────────────────────
   NO wrapper classes. NO mutable var score. NO applyBackTo().
   withPredictionScore() returns an immutable copy (Kotlin idiom).
  ─────────────────────────────────────────────────────────────────────
```

---

## Diagram 4: The Seams — Where We Cut

```
  reOrderGlobalEntitiesV2()                            ◄── ENTRY
    └── rankAndDedupeContent()
          └── rankAndMergeContent()
                └── rankContent()   ◄────────────────────── VERTICAL INCISION POINT
                      └── BaseEntityRankerConfiguration.rank()
                            │
            ┌───────────────┼───────────────────────────┐
            │               │                           │
            ▼               ▼                           ▼
     getEntities()    getScoreBundle()          getBoostBundle()
     ┄┄┄┄┄┄┄┄┄┄┄┄    ┄┄┄┄┄┄┄┄┄┄┄┄┄┄          + getRankingBundle()
     Flatten 9 types  Sibyl ML scoring         + getRankableContent()
     via type-checks  + blendBundle()          ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄
     ┄┄┄┄┄┄┄┄┄┄┄┄    (calibration,            Score assign, position
                       intent, boost,           boost, deal multiplier,
                       diversity rerank)        pin/flow sort, reassembly
            │               │                           │
            │               │                           │
            ▼               ▼                           ▼
  ╔═════════════════════════════════════════════════════════════════╗
  ║                                                                ║
  ║   PHASE 1: Single RANK_ALL step wraps ALL of this             ║
  ║   VerticalRankAllStep delegates to RankerConfiguration.rank() ║
  ║                                                                ║
  ╚═════════════════════════════════════════════════════════════════╝
            │               │                           │
            ▼               ▼                           ▼
  ╔═════════════════════════════════════════════════════════════════╗
  ║                                                                ║
  ║   PHASE 2: Decompose into granular steps                      ║
  ║                                                                ║
  ║   MODEL_SCORING ─► MULTIPLIER_BOOST ─► DIVERSITY_RERANK       ║
  ║        ─► POSITION_BOOSTING ─► FIXED_PINNING                  ║
  ║                                                                ║
  ╚═════════════════════════════════════════════════════════════════╝

  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄
  DefaultHomePageStoreRanker.rank()                ◄── HORIZONTAL INCISION
    └── modifyLiteStoreCollection()
          │
          ├── scoredCollections()              ─┐
          │   (Sibyl gRPC + modifiers)          │
          │                                     ├── Phase 1: RANK_ALL
          └── when(rankingType) { ... }         │   wraps all of this
              (30+ sort variants)              ─┘
```

---

## Diagram 5: The Four Interfaces

```
  ╔══════════════════════════════════════════════════════════════════╗
  ║                                                                 ║
  ║   INTERFACE 1: Scorable                                         ║
  ║   ─────────────────────                                         ║
  ║   "What gets ranked" — implemented directly by domain types     ║
  ║                                                                 ║
  ║   ┌──────────────────────────────────────────┐                  ║
  ║   │  <<interface>> Scorable                  │                  ║
  ║   ├──────────────────────────────────────────┤                  ║
  ║   │  fun scorableId(): String            ◄───┤── identity       ║
  ║   │  val predictionScore: Double?        ◄───┤── immutable val  ║
  ║   │  fun withPredictionScore(d): Scorable◄───┤── returns COPY   ║
  ║   └──────────────────────────────────────────┘                  ║
  ║        ▲    ▲    ▲    ▲    ▲    ▲    ▲    ▲    ▲                ║
  ║        │    │    │    │    │    │    │    │    │                 ║
  ║       SC   IC   DC  SCo  CV2  ICo  MC   RC   SE                ║
  ║                                                                 ║
  ║       9 domain types add `override` + one-line copy()           ║
  ║       NO wrapper classes. NO mutable state.                     ║
  ║                                                                 ║
  ╠══════════════════════════════════════════════════════════════════╣
  ║                                                                 ║
  ║   INTERFACE 2: RankingStep<S : Enum<S>>                         ║
  ║   ─────────────────────────────────────                         ║
  ║   "How ranking works" — generic over step type enum             ║
  ║                                                                 ║
  ║   ┌────────────────────────────────────────────────────────┐    ║
  ║   │  <<interface>> RankingStep<S : Enum<S>>                │    ║
  ║   ├────────────────────────────────────────────────────────┤    ║
  ║   │  val stepType: S                                       │    ║
  ║   │  suspend fun execute(items: List<Scorable>,            │    ║
  ║   │                      context: RankingContext)           │    ║
  ║   │    : List<Scorable>                                    │    ║
  ║   └────────────────────────────────────────────────────────┘    ║
  ║        ▲                                                        ║
  ║        │                                                        ║
  ║   ┌────┴──────────────────────────────────────────────┐         ║
  ║   │  VerticalRankAllStep                              │         ║
  ║   │  ─────────────────────────────────────────────────│         ║
  ║   │  stepType = VerticalStepType.RANK_ALL             │         ║
  ║   │  delegates to: RankerConfiguration.rank()         │         ║
  ║   │  (the entire legacy pipeline — zero behavior chg) │         ║
  ║   └───────────────────────────────────────────────────┘         ║
  ║                                                                 ║
  ║   Items in → items out. Pure function. No mutable state.        ║
  ║   Step type enum gives compile-time layer safety.               ║
  ║                                                                 ║
  ╠══════════════════════════════════════════════════════════════════╣
  ║                                                                 ║
  ║   INTERFACE 3: RankingHandler (Chain of Responsibility)         ║
  ║   ─────────────────────────────────────────────────────         ║
  ║   "Infrastructure wrapping" — metrics, conditions, shadow       ║
  ║                                                                 ║
  ║   ┌────────────────────────────────────────────────────────┐    ║
  ║   │  <<interface>> RankingHandler                          │    ║
  ║   │  suspend fun handle(items, context): List<Scorable>    │    ║
  ║   └────────────────────────────────────────────────────────┘    ║
  ║        ▲                                                        ║
  ║        │                                                        ║
  ║   ┌────┴──────────────┐                                         ║
  ║   │  BaseHandler      │                                         ║
  ║   │  + next: Handler? │──── chains to next handler              ║
  ║   └────┬──────────────┘                                         ║
  ║        │                                                        ║
  ║   ┌────┴──────────────┐                                         ║
  ║   │  StepHandler<S>   │                                         ║
  ║   │  wraps RankingStep│──── calls step.execute(), then next     ║
  ║   └───────────────────┘                                         ║
  ║                                                                 ║
  ║   Steps don't know they're in a chain.                          ║
  ║   Steps don't know they're being timed or traced.               ║
  ║                                                                 ║
  ╠══════════════════════════════════════════════════════════════════╣
  ║                                                                 ║
  ║   INTERFACE 4: Ranker<S : Enum<S>>                              ║
  ║   ────────────────────────────────                              ║
  ║   "Who orchestrates" — builds chain, dispatches                 ║
  ║                                                                 ║
  ║   ┌────────────────────────────────────────────────────────┐    ║
  ║   │  Ranker<S : Enum<S>>                                   │    ║
  ║   ├────────────────────────────────────────────────────────┤    ║
  ║   │  stepRegistry: Map<S, RankingStep<S>>                  │    ║
  ║   │                                                        │    ║
  ║   │  rank(items, stepTypes, context) → List<Scorable>      │    ║
  ║   │    1. buildChain(stepTypes) → RankingHandler           │    ║
  ║   │    2. chain.handle(items, context)                     │    ║
  ║   └────────────────────────────────────────────────────────┘    ║
  ║                                                                 ║
  ║   Zero business logic. Pure dispatch.                           ║
  ║   Same engine for vertical AND horizontal — only the            ║
  ║   step type enum differs.                                       ║
  ║                                                                 ║
  ╚══════════════════════════════════════════════════════════════════╝
```

---

## Diagram 6: Engine Dispatch — Handler Chain (Step by Step)

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                Ranker<VerticalStepType>.rank()                    │
  └──────────────────────────────────────────────────────────────────┘

  Input:                                     Step Types:
  List<Scorable>                             [RANK_ALL]
  (9 carousel types,                              │
   all implementing Scorable)                     │
           │                                      │
           │                                      ▼
           │                              ┌───────────────────────┐
           │                              │ buildChain([RANK_ALL])│
           │                              │                       │
           │                              │ Looks up each step    │
           │                              │ type in registry,     │
           │                              │ wraps in StepHandler, │
           │                              │ links via .next       │
           │                              └───────────┬───────────┘
           │                                          │
           ▼                                          ▼
  ═══════════════════════════════════════════════════════════════════

  HANDLER CHAIN (Phase 1: single link)

  ┌─────────────────────────────────────────────────────────────┐
  │  StepHandler<VerticalStepType>                              │
  │                                                             │
  │  wraps: VerticalRankAllStep                                 │
  │    stepType = RANK_ALL                                      │
  │    delegates to → RankerConfiguration.rank()                │
  │                                                             │
  │  This is the ENTIRE legacy pipeline:                        │
  │    getEntities() → getScoreBundle() → getBoostBundle()      │
  │    → getRankingBundle() → getRankableContent()              │
  │                                                             │
  │  .next = null (end of chain)                                │
  └─────────────────────────────────────────────────────────────┘
           │
           ▼
  Output: List<Scorable> (ranked)

  ═══════════════════════════════════════════════════════════════════

  HANDLER CHAIN (Phase 2: granular — future)

  ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
  │  StepHandler     │     │  StepHandler     │     │  StepHandler     │
  │  MODEL_SCORING   │────►│  MULTIPLIER_     │────►│  DIVERSITY_      │
  │                  │     │  BOOST           │     │  RERANK          │
  └──────────────────┘     └──────────────────┘     └──────────────────┘
                                                           │
                           ┌──────────────────┐     ┌──────┴───────────┐
                           │  StepHandler     │     │  StepHandler     │
                           │  FIXED_          │◄────│  POSITION_       │
                           │  PINNING         │     │  BOOSTING        │
                           └──────────────────┘     └──────────────────┘
```

---

## Diagram 7: Vertical + Horizontal Side by Side (Same Generic Engine)

```
  ┌─────────────── VERTICAL ──────────────┐   ┌────────────── HORIZONTAL ─────────────┐
  │  Ranking carousels on the page         │   │  Ranking stores within a carousel     │
  │                                        │   │                                       │
  │  INCISION POINT:                       │   │  INCISION POINT:                      │
  │  PostProcessor                         │   │  StoreRanker                           │
  │    .rankContent()                      │   │    .modifyLiteStoreCollection()        │
  │                                        │   │                                       │
  │  INTERFACE:                            │   │  INTERFACE:                            │
  │  ┌───────────────────────────┐         │   │  ┌───────────────────────────┐         │
  │  │  Scorable                 │         │   │  │  Scorable                 │         │
  │  │  (same interface)         │         │   │  │  (same interface)         │         │
  │  │  9 vertical types         │         │   │  │  3 horizontal types       │         │
  │  └───────────────────────────┘         │   │  └───────────────────────────┘         │
  │                                        │   │                                       │
  │  ENGINE:                               │   │  ENGINE:                               │
  │  ┌───────────────────────────┐         │   │  ┌───────────────────────────┐         │
  │  │ Ranker<VerticalStepType>  │         │   │  │ Ranker<HorizontalStepType>│         │
  │  │ (same Ranker class)       │         │   │  │ (same Ranker class)       │         │
  │  └───────────────────────────┘         │   │  └───────────────────────────┘         │
  │                                        │   │                                       │
  │  STEP (Phase 1):                       │   │  STEP (Phase 1):                      │
  │  ┌───────────────────────────┐         │   │  ┌───────────────────────────┐         │
  │  │  VerticalRankAllStep      │         │   │  │  HorizontalRankAllStep    │         │
  │  │  wraps: RankerConfig      │         │   │  │  wraps: modifyLiteStore   │         │
  │  │    .rank()                │         │   │  │    Collection()           │         │
  │  └───────────────────────────┘         │   │  └───────────────────────────┘         │
  │                                        │   │                                       │
  │  STEPS (Phase 2):                      │   │  STEPS (Phase 2):                     │
  │  ┌───────────────────────────┐         │   │  ┌───────────────────────────┐         │
  │  │  MODEL_SCORING            │         │   │  │  Granular steps TBD       │         │
  │  │  MULTIPLIER_BOOST         │         │   │  │  (mirrors vertical        │         │
  │  │  DIVERSITY_RERANK         │         │   │  │   pattern once proven)    │         │
  │  │  POSITION_BOOSTING        │         │   │  │                           │         │
  │  │  FIXED_PINNING            │         │   │  │                           │         │
  │  └───────────────────────────┘         │   │  └───────────────────────────┘         │
  │                                        │   │                                       │
  └────────────────────────────────────────┘   └───────────────────────────────────────┘

  Same Scorable. Same Ranker<S>. Same RankingStep<S>.
  Only the step type enum and step implementations differ.
  Vertical proven first → horizontal mirrors it.
```

---

## Diagram 8: Design Patterns Map

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                       DESIGN PATTERNS IN USE                        │
  └──────────────────────────────────────────────────────────────────────┘


  ① INTERFACE INHERITANCE              ② STRATEGY
  ──────────────────────────           ─────────────────────
  Domain types implement               Interchangeable step
  Scorable directly — no               algorithms plugged in
  wrappers                             via registry

  StoreCarousel : Scorable             ┌─ VerticalRankAllStep  (Phase 1)
  ItemCarousel  : Scorable             │
  DealCarousel  : Scorable             ├─ ModelScoringStep     (Phase 2)
  MapCarousel   : Scorable             ├─ MultiplierBoostStep  (Phase 2)
  ReelsCarousel : Scorable             ├─ DiversityRerankStep  (Phase 2)
  ... (9 total) : Scorable             ├─ PositionBoostStep    (Phase 2)
                                       └─ FixedPinningStep     (Phase 2)
  Fields already exist.
  Just add `override`.                 Engine picks from stepTypes list.


  ③ CHAIN OF RESPONSIBILITY            ④ FACADE
  ─────────────────────────            ─────────────────────
  Handlers linked via .next            One call hides all complexity
  Steps wrapped transparently

  items ──► [StepHandler] ──► [StepHandler] ──► ...    Ranker.rank(items, stepTypes, ctx)
               │                  │                       └─ buildChain(stepTypes)
               ▼                  ▼                       └─ chain.handle(items, ctx)
            step.execute()     step.execute()             └─ return result
            then next          then next


  ⑤ TEMPLATE METHOD                    Replaced by ① + ② + ③
  ─────────────────────
  THE PATTERN BEING REPLACED

  BaseEntityRankerConfiguration
      .rank()
       ├── getEntities()        ← rigid
       ├── getScoreBundle()     ← rigid
       ├── getBoostBundle()     ← rigid
       └── getRankableContent() ← rigid

  Fixed inheritance skeleton
  → replaced by config-driven
    handler chain
```

---

## Diagram 9: Shadow Validation → Rollout (Safe Delivery)

```
  ┌────────────────────────────────────────────────────────────────┐
  │                PHASE 1: SHADOW MODE                            │
  │                "Prove it's safe"                               │
  │                DV: ubp_shadow_vertical_ranking                 │
  └────────────────────────────────────────────────────────────────┘

                          Request
                             │
                             ▼
                  ┌─────────────────────┐
                  │ PostProcessor       │
                  │   .rankContent()    │
                  └──────────┬──────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
            ┌──────────────┐   ┌──────────────────────┐
            │   OLD PATH   │   │  ubpShadowEnabled?   │
            │  RankerConfig│   └──────────┬───────────┘
            │    .rank()   │        YES   │       NO
            │  (always     │              ▼        ▼
            │   runs)      │   ┌───────────────┐  SKIP
            │              │   │ Ranker<V>      │
            │              │   │  .rank(items,  │
            │              │   │   [RANK_ALL],  │
            │              │   │   ctx)         │
            │              │   │  (coroutine)   │
            │              │   └───────┬───────┘
            │              │           │
            ▼              │           ▼
     ┌──────────────┐     │   ┌───────────────┐
     │ Result → User │     │   │   COMPARE     │
     └──────────────┘     │   │   sort orders  │
                          │   │   log diffs    │
                          │   │   DISCARD      │
                          │   └───────────────┘
                          │
          Old path ALWAYS returns the result.
          Shadow NEVER affects the user.

  ─────────────────────────────────────────────────────────────────
         divergence_count = 0   (sustained across traffic)
  ─────────────────────────────────────────────────────────────────
                             │
                             ▼

  ┌────────────────────────────────────────────────────────────────┐
  │                PHASE 2: ROLLOUT MODE                           │
  │                "Migrate traffic"                               │
  └────────────────────────────────────────────────────────────────┘

                          Request
                             │
                             ▼
                  ┌─────────────────────┐
                  │ PostProcessor       │
                  │   .rankContent()    │
                  └──────────┬──────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
               ubpRolloutEnabled?     │
                    │                 │
               YES  │            NO   │
                    ▼                 ▼
            ┌──────────────┐   ┌──────────────┐
            │ Ranker<V>    │   │  OLD PATH    │
            │  .rank()     │   │  RankerConfig │
            │  (primary)   │   │  (unchanged) │
            └──────┬───────┘   └──────┬───────┘
                   │                  │
                   ▼                  ▼
            Result → User       Result → User

             Ramp: 1% ──► 5% ──► 25% ──► 50% ──► 100%

             Rollback = disable DV. Immediate. No deploy.
```

---

## Diagram 10: The Full Pipeline (4 Layers)

```
                              REQUEST
                                 │
                                 ▼
  ╔══════════════════════════════════════════════════════════════════╗
  ║  LAYER 1: RETRIEVAL                                             ║
  ║  DefaultHomePageStoreContentFetching.fetch()                    ║
  ║  Fetch ~200-1000 candidate stores, items, carousels             ║
  ╚══════════════════════════════╤═══════════════════════════════════╝
                                 │
                                 ▼
  ╔══════════════════════════════════════════════════════════════════╗
  ║  LAYER 2: GROUPING                                              ║
  ║  DefaultHomePageStoreContentGrouper.group()                     ║
  ║  Bucket into ~20 named carousels (Favorites, Taste, NV, ...)   ║
  ╚══════════════════════════════╤═══════════════════════════════════╝
                                 │
                                 ▼
  ╔══════════════════════════════════════════════════════════════════╗
  ║  LAYER 3: HORIZONTAL RANKING                    ◄── UBP SCOPE  ║
  ║  ┌───────────────────────────────────────────┐                  ║
  ║  │  DefaultHomePageStoreRanker.rank()        │                  ║
  ║  │  Within each carousel:                    │                  ║
  ║  │  rank stores/items by ML score            │                  ║
  ║  │                                           │                  ║
  ║  │  NEW: Ranker<HorizontalStepType>          │                  ║
  ║  │       + HorizontalRankAllStep             │                  ║
  ║  └───────────────────────────────────────────┘                  ║
  ╚══════════════════════════════╤═══════════════════════════════════╝
                                 │
                        ┌────────┴────────────┐
                        │                     │
                        ▼                     ▼
  ╔═══════════════════════════╗  ╔═══════════════════════════════════╗
  ║ LAYER 4: VERTICAL RANKING║  ║  ICP ADS HORIZONTAL BLENDING      ║
  ║          ◄── UBP SCOPE   ║  ║  (SEPARATE SYSTEM — NOT UBP)      ║
  ║ ┌───────────────────────┐║  ║                                   ║
  ║ │ PostProcessor         │║  ║  HomepageAdsBlender.blend()       ║
  ║ │   .rankContent()      │║  ║  Runs in PARALLEL to Layer 4      ║
  ║ │                       │║  ║                                   ║
  ║ │ NEW: Ranker<Vertical  │║  ║  (Phase 3: make organic + ads     ║
  ║ │   StepType>           │║  ║   compete on same value scale)    ║
  ║ │ + VerticalRankAllStep │║  ║                                   ║
  ║ └───────────────────────┘║  ║                                   ║
  ╚════════════╤══════════════╝  ╚═════════════════╤═════════════════╝
               │                                   │
               └──────────────┬────────────────────┘
                              │
                              ▼
                        SERIALIZE → CLIENT
```

---

## Diagram 11: Before vs After — Adding a New Carousel Type

```
  ┌──────────────────────── BEFORE ────────────────────────────────┐
  │                                                                │
  │  New carousel type "RecommendationCarousel"                    │
  │                                                                │
  │  Touch list:                                                   │
  │   [1] EntityRankerConfiguration.getEntities()     + branch     │
  │   [2] EntityRankerConfiguration.getScoreBundle()  + branch     │
  │   [3] BlendingUtil.blendBundle()                  + branch     │
  │   [4] BoostingBundle logic                        + branch     │
  │   [5] EntityRankerConfiguration.getRankingBundle()+ branch     │
  │   [6] EntityRankerConfiguration.getRankable...()  + branch     │
  │   [7] PostProcessor scoring integration           + branch     │
  │   [8] PostProcessor dedup logic                   + branch     │
  │   [9] StoreRanker horizontal wiring               + branch     │
  │  [10] Layout serialization                        + branch     │
  │  [11] ...more                                     + branch     │
  │                                                                │
  │  10+ files changed. 2-3 weeks. "Edit and pray."               │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘


  ┌──────────────────────── AFTER ─────────────────────────────────┐
  │                                                                │
  │  New carousel type "RecommendationCarousel"                    │
  │                                                                │
  │   ┌──────────────────────────────────────────────────┐         │
  │   │  data class RecommendationCarousel(              │         │
  │   │      val id: String,                             │         │
  │   │      override val predictionScore: Double?,      │         │
  │   │      // ... other fields ...                     │         │
  │   │  ) : Scorable {                                  │         │
  │   │      override fun scorableId() = id              │         │
  │   │      override fun withPredictionScore(score) =   │         │
  │   │          copy(predictionScore = score)            │         │
  │   │  }                                               │         │
  │   └──────────────────────────────────────────────────┘         │
  │                                                                │
  │  Add `override` to existing fields + one-line copy().          │
  │  Pipeline, engine, steps — UNCHANGED.                          │
  │  No wrapper class. No adapter file.                            │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘
```

---

## Diagram 12: The Value Function

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │   EV(component, position) = pImp(k) × pAct(c) × vAct(c)        │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘

       ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
       │     pImp(k)       │  │     pAct(c)       │  │     vAct(c)       │
       │                  │  │                  │  │                  │
       │  P(user sees     │  │  P(user acts |   │  │  Value of that   │
       │  position k)     │  │  they see c)     │  │  action          │
       │                  │  │                  │  │                  │
       │  Position decay  │  │  ML model output │  │  gov_w × GOV     │
       │  Row 0 > Row 3   │  │  (Sibyl)         │  │  + fiv_w × FIV   │
       │  Slot 0 > Slot 5 │  │  CTR, CVR, order │  │  + strat_w ×     │
       │                  │  │  probability     │  │    Strategic     │
       │  Owner: BE       │  │  Owner: MLE      │  │  Owner: MLE      │
       └──────────────────┘  └──────────────────┘  └──────────────────┘
              │                      │                      │
              ▼                      ▼                      ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                PHASE 1 (SHIPPED)                                 │
  │                                                                  │
  │   RANK_ALL step wraps entire pipeline — scoring, blending,       │
  │   boosting, pinning all in one step.                             │
  │   Value function is implicit in existing BlendingUtil logic.     │
  │                                                                  │
  ├──────────────────────────────────────────────────────────────────┤
  │                PHASE 2 (PLANNED) — Granular Decomposition       │
  │                                                                  │
  │   [Step] MODEL_SCORING      → pure pAct (Sibyl prediction)      │
  │   [Step] MULTIPLIER_BOOST   → deal/position multipliers         │
  │   [Step] DIVERSITY_RERANK   → diversity constraints             │
  │   [Step] POSITION_BOOSTING  → position-based score adjustments  │
  │   [Step] FIXED_PINNING      → hardcoded position enforcement    │
  │                                                                  │
  ├──────────────────────────────────────────────────────────────────┤
  │                FUTURE — Additional Steps                        │
  │                                                                  │
  │   [Step] CALIBRATION        → normalize scores across types     │
  │   [Step] VALUE_WEIGHTING    → explicit vAct weights             │
  │   [Step] POSITION_DECAY     → apply pImp(k)                    │
  │                                                                  │
  │   No engine change. Just register new step types + impls.       │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Diagram 13: The N × M Experiment Model

```
  ┌─────────────────────────────────────────────────────────────────┐
  │               EXPERIMENT INDEPENDENCE BY LAYER                  │
  └─────────────────────────────────────────────────────────────────┘

                     Horizontal Experiments
                  (within-carousel store ordering)

                  Control    Treatment A   Treatment B
                ┌──────────┬──────────────┬──────────────┐
  Vertical    C │          │              │              │
  Experiments o │  C × C   │   C × A      │   C × B     │
  (carousel   n │          │              │              │
   ordering)  t ├──────────┼──────────────┼──────────────┤
              r │          │              │              │
            T o │  T1 × C  │   T1 × A     │   T1 × B    │
            r l │          │              │              │
            e   ├──────────┼──────────────┼──────────────┤
            a   │          │              │              │
            t T │  T2 × C  │   T2 × A     │   T2 × B    │
            . 2 │          │              │              │
              . └──────────┴──────────────┴──────────────┘

  RULES:
  ─────
  • Mutually exclusive WITHIN a layer
    (user gets exactly 1 horizontal + exactly 1 vertical assignment)

  • Independent ACROSS layers
    (any horizontal × vertical combination is valid)

  • N horizontal × M vertical = N×M experiment cells
```

---

## Diagram 14: MLE Experiment Workflow — Before vs After

```
  ┌───────────────────── BEFORE ───────────────────────────────────┐
  │                                                                │
  │  MLE wants to test a new model for vertical ranking:           │
  │                                                                │
  │   1. File Jira ticket to HP team                               │
  │   2. HP eng reads 6+ files to understand current flow          │
  │   3. HP eng modifies DV keys (2-3 files)                       │
  │   4. HP eng modifies runtime JSON configs (3-4 files)          │
  │   5. HP eng adds Sibyl predictor wiring (1-2 files)            │
  │   6. HP eng adds logging / instrumentation (1-2 files)         │
  │   7. Code review cycle (1+ week)                               │
  │   8. Deploy, ramp, monitor                                     │
  │                                                                │
  │   Timeline: 2-3 WEEKS                                          │
  │   Files touched: 10-15                                         │
  │   People blocked: MLE waiting on HP eng                        │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘


  ┌───────────────────── AFTER ────────────────────────────────────┐
  │                                                                │
  │  MLE wants to test a new model for vertical ranking:           │
  │                                                                │
  │   1. Write JSON config:                                        │
  │      ┌──────────────────────────────────────────────┐          │
  │      │  "treatment_new_model": {                    │          │
  │      │    "extends": "control",                     │          │
  │      │    "p_act": {                                │          │
  │      │      "model": "store_ranker_v2"              │          │
  │      │    }                                         │          │
  │      │  }                                           │          │
  │      └──────────────────────────────────────────────┘          │
  │                                                                │
  │   2. Add one DV key                                            │
  │                                                                │
  │   3. Deploy (hot-reload, no pod restart)                       │
  │                                                                │
  │   Timeline: 2-3 DAYS                                           │
  │   Files touched: 1-2                                           │
  │   People blocked: NONE — self-service                          │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘
```

---

## Diagram 15: Strangler Fig — Phase 1 → Phase 2 Evolution

```
                          ┌────────────────┐
                          │    REQUEST     │
                          └───────┬────────┘
                                  │
                                  ▼
                          ┌───────────────┐
                          │  DV GATE      │
                          └───┬───────┬───┘
                              │       │
                   UBP ON     │       │    UBP OFF
                              │       │
                    ┌─────────▼──┐  ┌─▼──────────┐
                    │            │  │             │
                    │  ┌──────┐  │  │  ┌───────┐  │
                    │  │ NEW  │  │  │  │  OLD  │  │
                    │  │ PATH │  │  │  │  PATH │  │
                    │  │      │  │  │  │       │  │
                    │  │Ranker│  │  │  │ Base  │  │
                    │  │  <V> │  │  │  │Entity │  │
                    │  │.rank │  │  │  │Ranker │  │
                    │  │  ()  │  │  │  │Config │  │
                    │  │      │  │  │  │.rank()│  │
                    │  └──┬───┘  │  │  └──┬────┘  │
                    │     │      │  │     │       │
                    └─────┼──────┘  └─────┼───────┘
                          │               │
                          ▼               ▼
                     Result → User   Result → User

  ─────────────────────────────────────────────────────────────
   Both paths COMPILE and RUN at every stage.
   Old path is NEVER modified.
   Rollback = flip DV. Instant. No deploy.
  ─────────────────────────────────────────────────────────────

   EVOLUTION OVER TIME:

   Phase 1    [RANK_ALL]                          ← wraps entire legacy
              (single step, proves interfaces)

   Phase 2    [MODEL_SCORING] ─► [MULTIPLIER_BOOST] ─► [DIVERSITY_RERANK]
                  ─► [POSITION_BOOSTING] ─► [FIXED_PINNING]
              (5 granular steps, each independently testable)

   Future     [MODEL_SCORING] ─► [CALIBRATION] ─► [VALUE_WEIGHTING]
                  ─► [DIVERSITY] ─► [POSITION_DECAY]
              (value function steps, MLE self-service)

   ENGINE NEVER CHANGES. Just add enum values + step implementations.

  ─────────────────────────────────────────────────────────────

   ROLLOUT RAMP:

   Week 1     ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  0% UBP
   Week 2     █░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  1% UBP
   Week 3     █████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  5% UBP
   Week 4     ██████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 25% UBP
   Week 5     ████████████████████░░░░░░░░░░░░░░░░░░░░░░ 50% UBP
   Week 6     ████████████████████████████████████████████ 100% UBP
```

---

## Diagram 16: Immutability — Old Design vs New Design

```
  ┌──────────────────── OLD DESIGN (rejected) ─────────────────────┐
  │                                                                │
  │   class StoreCarouselRow(val carousel: StoreCarousel) : FeedRow│
  │       var score: Double = 0.0       ← MUTABLE                 │
  │       val metadata: MutableMap      ← MUTABLE                 │
  │       fun applyBackTo()             ← writes back to domain   │
  │                                                                │
  │   9 wrapper classes. Mutable state. Write-back ceremony.       │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘

                              vs

  ┌──────────────────── NEW DESIGN (shipped) ──────────────────────┐
  │                                                                │
  │   data class StoreCarousel(                                    │
  │       override val predictionScore: Double?   ← IMMUTABLE val │
  │   ) : Scorable {                                               │
  │       override fun withPredictionScore(score) =                │
  │           copy(predictionScore = score)        ← returns COPY  │
  │   }                                                            │
  │                                                                │
  │   0 wrapper classes. Immutable. Kotlin idiomatic copy().       │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘

  ┌─────────────────── DATA FLOW COMPARISON ───────────────────────┐
  │                                                                │
  │  OLD: domain obj → wrap in FeedRow → mutate .score             │
  │       → applyBackTo() → domain obj updated                    │
  │                                                                │
  │  NEW: domain obj IS Scorable → withPredictionScore()           │
  │       → NEW immutable copy → collect in list                  │
  │       → toRankableContent() → typed container                 │
  │                                                                │
  │  No mutation. No writeback. Round-trip via conversion fns.     │
  └────────────────────────────────────────────────────────────────┘
```

---

## Diagram 17: Phase 1 → Phase 2 Step Decomposition

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  PHASE 1 (SHIPPED): One step wraps everything                   │
  └──────────────────────────────────────────────────────────────────┘

  List<Scorable>
       │
       ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  RANK_ALL                                                    │
  │  ┌────────────────────────────────────────────────────────┐  │
  │  │  getEntities() → getScoreBundle() → getBoostBundle()  │  │
  │  │  → getRankingBundle() → getRankableContent()           │  │
  │  │                                                        │  │
  │  │  All entangled inside RankerConfiguration.rank()       │  │
  │  └────────────────────────────────────────────────────────┘  │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
       │
       ▼
  List<Scorable> (ranked — identical to old path)


  ┌──────────────────────────────────────────────────────────────────┐
  │  PHASE 2 (PLANNED): Granular, composable, independently testable│
  └──────────────────────────────────────────────────────────────────┘

  List<Scorable>
       │
       ▼
  ┌───────────────────┐
  │  MODEL_SCORING    │  Sibyl gRPC → predictionScore
  │  (sets pAct)      │
  └─────────┬─────────┘
            │
            ▼
  ┌───────────────────┐
  │  MULTIPLIER_BOOST │  calibration × intent × boost weights
  │                   │
  └─────────┬─────────┘
            │
            ▼
  ┌───────────────────┐
  │  DIVERSITY_RERANK │  diversity constraints, category spread
  │                   │
  └─────────┬─────────┘
            │
            ▼
  ┌───────────────────┐
  │ POSITION_BOOSTING │  position-based score adjustments
  │                   │
  └─────────┬─────────┘
            │
            ▼
  ┌───────────────────┐
  │  FIXED_PINNING    │  hardcoded position enforcement
  │                   │
  └─────────┬─────────┘
            │
            ▼
  List<Scorable> (ranked — same result, decomposed path)

  ─────────────────────────────────────────────────────────────
  Engine is UNCHANGED between Phase 1 and Phase 2.
  Just add enum values + step implementations.
  Each step independently testable and configurable.
  ─────────────────────────────────────────────────────────────
```
