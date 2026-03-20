# Unified Blending Platform — Engineering First Principles

> This is the northstar. When in doubt, return here.

---

## 0. Two Different Meanings of "Horizontal Blending"

Before anything else: "horizontal" in this codebase is overloaded.

| Term | Meaning | Pipeline stage | UBP phase |
|---|---|---|---|
| **Horizontal ranking** | Ordering stores within a carousel by ML score | Layer 3 — before post-processing | Phase 1-2 |
| **Horizontal blending** | Inserting ad store cards into ranked organic lists | Parallel to Layer 4 — ICP ads system | Phase 3 |

These are implemented by entirely separate systems. The ICP ads blender (`HomepageAdsBlender`) runs in a parallel branch to post-processing and merges its output just before serialization. UBP Phase 1-2 does not touch it.

---

## 1. The Homepage Is a Grid

```
                  horizontal →
         slot 0      slot 1      slot 2      slot 3
        ┌───────────────────────────────────────────┐
row 0   │ [store A]  [store B]  [store C]  [store D] │  ← Favorites carousel
        ├───────────────────────────────────────────┤
row 1   │ [store E]  [store F]  [store G]  [store H] │  ← Taste carousel
        ├───────────────────────────────────────────┤
row 2   │ [item X]   [item Y]   [item Z]             │  ← Item carousel
        ├───────────────────────────────────────────┤
row 3   │ [store I]  [store J]                       │  ← NV carousel
        └───────────────────────────────────────────┘
  ↑
vertical
```

Two independent ranking problems, sequentially executed:

- **Vertical ranking**: which row (carousel) goes at which position?
- **Horizontal ranking**: within a row, which stores/items go in which slots?

That's the whole model. Everything else is implementation detail.

---

## 2. The Full Pipeline (What Actually Runs)

```
REQUEST
   │
   ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 1: RETRIEVAL                                      │
│  Fetch candidate stores, items, carousels from upstream  │
│  Classes: restaurantStoreContentRetrievalJob             │
│  Output: raw pool of ~200-1000 candidates                │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 2: GROUPING                                       │
│  Bucket candidates into named carousels                  │
│  Classes: restaurantStoreGroupingJob                     │
│  (Favorites, Taste, Reorder, NV, etc.)                   │
│  Output: ~20 carousels, each with ~10-50 candidates      │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 3: HORIZONTAL RANKING                             │
│  Within each carousel: rank stores/items by ML score     │
│  Classes: DefaultHomePageStoreRanker                     │
│           DefaultHomePageCampaignRanker                  │
│  Output: same carousels, stores now in ranked order      │
│                                                          │
│  ⚠ S1/S2/S3 decoration reranking runs AFTER Iguazu      │
│    fires → logged positions ≠ positions users see        │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYOUT PROCESSING                                       │
│  Assemble StoreCarousel objects for post-processing      │
└──────────────────────┬───────────────────────────────────┘
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
┌─────────────────────┐  ┌───────────────────────────────┐
│  LAYER 4: VERTICAL  │  │  ICP ADS HORIZONTAL BLENDING  │
│  RANKING            │  │  HomepageAdsBlender.blend()   │
│  reOrderGlobalEntities  │  4 parallel targets:         │
│  V2()               │  │  - store carousels (2D blend) │
│  UR scoring, multi- │  │  - home feed store list       │
│  pliers, dedup,     │  │  - PAD collections            │
│  fixups             │  │  - smart suggestions          │
│  Classes:           │  │                               │
│  DefaultHomePage    │  │  Algorithms:                  │
│  PostProcessor      │  │  GenericHorizontalBlenderV2   │
└──────────┬──────────┘  │  DedupeLastHorizontalBlender  │
           │             └──────────────┬────────────────┘
           └────────┬────────────────────┘
                    │
                    ▼
               SERIALIZE → CLIENT
```

**UBP governs layers 3 and 4.** Retrieval and grouping are upstream inputs. The ICP ads blending system is a separate parallel pipeline — UBP Phase 3 will eventually make organic and ad scores comparable so they can compete for the same slots.

---

## 3. One Value Function for Both Layers

Every ranking decision — whether ranking a store within a carousel or a carousel on the page — answers the same question:

> **What is the expected value of showing this component to this user at this position?**

```
Expected Value = pImp × pAct × vAct

  pImp = P(user sees this component at position k)
         Decays with position. Row 0 > Row 3. Slot 0 > Slot 5.

  pAct = P(user takes action | they see it)
         The ML model output. Predicted CTR, CVR, order probability.
         Must be calibrated to a comparable scale across all content types.

  vAct = Value of that action (in business terms)
         GOV (gross order value), FIV (first-impression value),
         ads revenue, strategic value (NV trial, retention).
         Weights are configurable per experiment.
```

**Why this matters:** Today organic, ads, merch, and NV each use different scoring functions on incomparable scales. You cannot rank them together. UBP's entire value proposition is making these comparable through calibration and an explicit value function.

---

## 4. The Two Ranking Engines

Both engines share the same shape: a config-driven pipeline of steps executed in sequence.

```
INPUT: list of Components (carousels OR stores, depending on layer)
           │
           ▼
    ┌─────────────────────────────┐
    │  for each step in config:   │
    │    processor = registry     │
    │                 [step.type] │
    │    processor.process(       │
    │      components,            │
    │      context,               │
    │      step.params            │   ← params come from config JSON, not from code
    │    )                        │
    │    if emitTrace:            │
    │      record score snapshot  │   ← automatic, no manual instrumentation
    └──────────────┬──────────────┘
                   │
                   ▼
OUTPUT: same Components, reordered by final score
```

**Vertical engine** (Layer 4): components are carousels. Steps: MODEL_SCORING, BOOST_AND_RANK.

**Horizontal engine** (Layer 3): components are stores/items. Steps: MODEL_SCORING, RANKING_SORT.

---

## 5. Three Core Interfaces

Everything in UBP is built on three interfaces. If you understand these, you understand the platform.

### Component — the unit being ranked

```kotlin
interface VerticalComponent {
    val id: String
    val type: ComponentType          // STORE_CAROUSEL, ITEM_CAROUSEL, NV_CAROUSEL, etc.
    var score: Double                // mutable — processors update this
    val metadata: MutableMap<String, Any>
    fun recordTrace(stepId: String, value: Any)
}
```

Today there are 9+ separate typed carousel classes with no shared interface. UBP collapses them to one.

### Processor — one step in the pipeline

```kotlin
interface VerticalProcessor {
    val type: String                 // "MODEL_SCORING", "BOOST_AND_RANK", etc.

    suspend fun process(
        components: MutableList<VerticalComponent>,
        context: RankingContext,
        params: Map<String, Any>,   // injected from config, not read internally
    )
}
```

The critical property: `params` are **injected at call time**, not pulled from DV keys inside the processor. This is what makes processors reusable across experiments.

### Engine — the orchestrator

```kotlin
class VerticalRankingEngine(
    val processorRegistry: Map<String, VerticalProcessor>,
) {
    suspend fun rank(
        components: MutableList<VerticalComponent>,
        config: PipelineConfig,
        context: RankingContext,
    ): List<VerticalComponent>
}
```

The engine has no business logic. It reads the step list from config and dispatches to processors. New processor = new entry in the registry. Changed pipeline = changed JSON config. No code changes to the engine.

---

## 6. The Experiment Model: N × M

```
N experiments on horizontal ranking
M experiments on vertical ranking
─────────────────────────────────────
Each user is assigned to exactly ONE horizontal experiment variant
Each user is assigned to exactly ONE vertical experiment variant
A user can be in ANY (horizontal, vertical) combination simultaneously

This gives N × M experiment cells.
Experiments are mutually exclusive WITHIN a layer, not ACROSS layers.
```

In practice, implemented as independent DV keys:
```
DV "ubp_hp_horizontal_v2" → "treatment_fw"    (horizontal assignment)
DV "ubp_hp_vertical_v3"   → "intent_model__v3" (vertical assignment)
```

Each DV is read independently. The two assignments don't interfere. The engine for each layer resolves its own config independently from its own DV value.

**This is already how DV works today.** UBP just formalizes it as a first-class property of the platform.

---

## 7. The Config Contract

One JSON file per experiment. One DV controls which variant runs. No code changes to run a new experiment variant.

```json
// ubp/experiments/{experiment_id}.json
{
  "control": {
    "vertical_pipeline": {
      "steps": [
        { "id": "score",          "type": "MODEL_SCORING",  "params": { "predictor_ref": "p_act" } },
        { "id": "boost_and_rank", "type": "BOOST_AND_RANK", "params": { ... } }
      ]
    }
  },
  "treatment_intent_v3": {
    "vertical_pipeline": {
      "steps": [
        { "id": "score",          "type": "MODEL_SCORING",  "params": { "model_name": "vertical_intent_v3" } },
        { "id": "boost_and_rank", "type": "BOOST_AND_RANK", "params": { ... } }
      ]
    }
  }
}
```

**MLE surface:** drop a JSON file, add one DV key. No PR for parameter changes.

**BE surface:** register a new Processor class when a new step type is needed. Everything else is config.

---

## 8. What "Unified" Means

"Unified" has three concrete meanings:

**1. Unified Component Model**
Every content type (store carousel, item carousel, NV carousel, ads carousel) implements the same `Component` interface. The ranking engine doesn't know or care what type of content it's ranking. There is no `if (component is StoreCarousel)` in the engine.

**2. Unified Value Function**
Every content type's score is expressed as `pImp × pAct × vAct` on a comparable, calibrated scale. Organic stores, ads, merch, and NV placements compete on the same scale. No content type gets a separate scoring system or a hardcoded multiplier that can't be measured.

**3. Unified Experiment Platform**
Both horizontal and vertical ranking are configured through the same JSON contract, the same DV wiring convention, and the same tracing infrastructure. MLEs self-serve both layers through the same interface. No layer requires a different workflow.

---

## 9. What the Current Code Gets Wrong

These are the root causes that UBP addresses:

| Root Cause | Symptom | UBP Fix |
|---|---|---|
| Each carousel type owns its own ranking logic | FavoritesCarousel, DVCarousel, TasteCarousel each do ranking differently, no shared interface | `Component` interface collapses all types; `Processor` registry handles step logic |
| Step params live in 10+ scattered places | Changing one experiment touches 5 DVs + 3 JSON files + hardcoded constants | Single experiment JSON per treatment; params injected at call time |
| No config seam for step sequence | Adding/removing a step requires code change | Steps declared in config; engine composes from registry at request time |
| Post-ranking fixups are outside the pipeline | `updateSortOrderOfNVCarousel()` runs after the DAG with no traceability | Business fixups stay outside pipeline; MLE-configurable pins are part of `BOOST_AND_RANK` |
| Manual per-step Iguazu instrumentation | Each processor manually calls IguazuEventUtil — inconsistent, often missing | Engine auto-calls `recordTrace()` after every step; standard schema |
| Organic/ads/NV scores are incomparable | Ads inserted AFTER organic ranking — highest-value ads may not be at the top | All content declares `pAct` + `vAct`; calibration service normalizes scores |

---

## 10. The Northstar: What Done Looks Like

```
An MLE who wants to run a new vertical blending experiment:

  1. Writes a JSON file:  ubp/experiments/hp_vertical_blend_v4.json
  2. Adds one DV key:     UBP_HP_VERTICAL_BLEND_V4("ubp_hp_vertical_blend_v4")
  3. Deploys: runtime JSON hot-reload, no pod restart

That's it. The BE engine handles:
  - Feature resolution (from config's feature declarations)
  - Sibyl RPC (from config's model name)
  - Calibration (from config's calibration spec)
  - Tracing (automatic per-step, standard schema)
  - Traffic splitting (via DV infrastructure)

What the MLE never touches:
  - Java/Kotlin source code
  - PR review cycles for parameter changes
  - Manual logging instrumentation
  - Cross-team alignment for DV wiring
```

The day that workflow is possible for both vertical and horizontal ranking, UBP is done.
