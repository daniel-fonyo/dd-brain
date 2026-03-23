# Unified Blending Platform — Engineering First Principles

> This is the northstar. When in doubt, return here.
> Naming follows `poc-generic-ranking.md`. Contract follows `mle-contract.md`.

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

Every ranking decision answers the same question:

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

**Variable dictionary:**

| Abbrev | Full name | What it is | Who owns it |
|---|---|---|---|
| `EV` | Expected Value | Final ranking score — higher = shown earlier | Engine output |
| `pImp(k)` | Probability of Impression | P(user sees component at position k). `decay_rate ^ k` | BE — not MLE-configurable |
| `pAct(c)` | Probability of Action | P(user clicks/orders given they see c). Sibyl model output. | MLE provides model + features |
| `vAct(c)` | Value of Action | Business worth if action happens: `gov_w × GOV + fiv_w × FIV + strategic_w × Strategic` | MLE provides weights |
| `GOV` | Gross Order Value | Revenue from the order | BE signal |
| `FIV` | First Impression Value | Strategic value of first-time NV/vertical exposure | BE signal |

Today `vAct` and `pImp` are implicit — boost weights stand in for value weights with no principled formula. Phase 1 makes `pAct` explicit. Future phases make `vAct` and `pImp` explicit.

---

## 4. Core Interfaces (from poc-generic-ranking.md)

### `Scorable` — the unit being ranked

```kotlin
interface Scorable {
    val id: String
    val predictionScore: Double?
    fun withPredictionScore(score: Double): Scorable
}
```

Not a wrapper — existing domain types implement it directly. 12 types already have `id` and `predictionScore`. Adding `Scorable` = adding `override` keywords + one-line `withPredictionScore` via `copy()`.

### `RankingStep<S>` — domain logic

```kotlin
interface RankingStep<S : Enum<S>> {
    val stepType: S
    suspend fun execute(items: List<Scorable>, context: RankingContext): List<Scorable>
}
```

Items in → items out. Pure function. No mutable state. Parameterized by step type enum for compile-time layer safety.

### `RankingHandler` — infrastructure (Chain of Responsibility)

```kotlin
interface RankingHandler {
    suspend fun handle(items: List<Scorable>, context: RankingContext): List<Scorable>
}
```

Wraps steps with cross-cutting concerns: metrics, conditions, shadow execution. Steps don't know they're being timed or traced.

### `Ranker<S>` — the engine

```kotlin
class Ranker<S : Enum<S>>(
    private val stepRegistry: Map<S, RankingStep<S>>,
    private val metrics: MetricsClient,
) {
    suspend fun rank(items: List<Scorable>, pipeline: List<S>, ctx: RankingContext): List<Scorable>
}
```

Assembles handler chain from pipeline config, executes it. Zero business logic. Same engine for vertical and horizontal — only the step enum and step implementation differ.

---

## 5. Design Patterns

| Pattern | Where | Role |
|---|---|---|
| **Strategy** | `RankingStep` implementations | Engine dispatches to whichever step the config specifies |
| **Chain of Responsibility** | `RankingHandler` chain | Steps linked via `next`; metrics/conditions wrap steps transparently |
| **Adapter** | `Scorable` on domain types | 12 domain types adapted to one interface; engine never type-checks |
| **Facade** | `Ranker.rank()` | Hides config resolution, handler chain assembly, tracing |
| **Template Method** (replaced) | `BaseEntityRankerConfiguration.rank()` | The current code's fixed skeleton — replaced by config-driven chain |

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

In practice, independent DV keys:
```
DV "ubp_hp_horizontal_v2" → "treatment_fw"       (horizontal assignment)
DV "ubp_hp_vertical_v3"   → "intent_model__v3"   (vertical assignment)
```

---

## 7. The Config Contract

See `mle-contract.md` for full schema and examples. Summary:

Step order is **hardcoded**: `MODEL_SCORING → BOOST_AND_RANK`. MLEs configure only the knobs.

```json
{
  "control": {
    "p_act": { "predictor": "feed_ranking_v1", "model": "store_ranker_v1", "features": [...] },
    "v_act": { "gov": 1.0, "fiv": 0.0, "strategic": 0.0 },
    "boost_weights": { "nv": 1.0 },
    "diversity": { "enabled": false, "weight": 0.0 }
  },
  "treatment_nv_boost": {
    "extends": "control",
    "v_act": { "gov": 0.8, "strategic": 0.2 },
    "boost_weights": { "nv": 1.5 }
  }
}
```

**MLE surface:** drop a JSON file, add one DV key. No PR for parameter changes.
**BE surface:** register a new `RankingStep` when a new step type is needed. Everything else is config.

---

## 8. What "Unified" Means

**1. Unified Interface Model**
Every content type implements `Scorable`. The engine doesn't know or care what type of content it's ranking. No `if (item is StoreCarousel)` in the engine.

**2. Unified Value Function**
Every content type's score expressed as `pImp × pAct × vAct` on a comparable, calibrated scale. Organic stores, ads, merch, and NV placements compete on the same scale.

**3. Unified Experiment Platform**
Both horizontal and vertical ranking configured through the same JSON contract, the same DV wiring, and the same tracing infrastructure.

---

## 9. What the Current Code Gets Wrong

| Root Cause | Symptom | UBP Fix |
|---|---|---|
| Each carousel type owns its own ranking logic | 9+ types with no shared interface | `Scorable` interface collapses all types |
| Step params live in 10+ scattered places | Changing one experiment touches 5 DVs + 3 JSON files | Single experiment JSON; params injected at call time |
| No config seam for step sequence | Adding/removing a step requires code change | Steps declared in config; engine composes from registry |
| Post-ranking fixups outside pipeline | `updateSortOrderOfNVCarousel()` runs after DAG with no traceability | Business fixups stay outside; MLE-configurable pins in step params |
| Manual Iguazu instrumentation | Each step manually calls IguazuEventUtil — inconsistent | Engine auto-traces after every step |
| Scores are incomparable | Ads inserted AFTER organic ranking on different scale | All content declares `pAct` + `vAct`; calibration normalizes |

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

---

## Horizontal Contract Notes (Phase 2)

Same formula, same pattern — applied to store/item ordering *within* each carousel.

Key difference: carousels have different ranking needs. The contract supports `carousel_overrides` — per-carousel step params on top of defaults.

| Current problem | UBP fix |
|---|---|
| Each carousel type owns ranking logic in `modifyLiteStoreCollection()` | `RankingStep` registry + `carousel_overrides` in config |
| S1/S2/S3 reranking invisible (runs after Iguazu fires) | Part of `RANKING_SORT` step, auto-traced |
| Same config fragmentation as vertical | Same `p_act` + `v_act` contract |
