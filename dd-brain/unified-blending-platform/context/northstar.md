# Unified Blending Platform — Engineering First Principles

> This is the northstar. When in doubt, return here.

---

## 0. Two Different Meanings of "Horizontal Blending"

Before anything else: there are **two different things** in the codebase both operating "horizontally":

```
ORGANIC HORIZONTAL RANKING          ADS HORIZONTAL BLENDING
(restaurantStoreRankingJob)         (HomepageAdsBlender, separate parallel job)
────────────────────────────────    ────────────────────────────────────────────
Ranks organic stores by ML          Inserts ad store cards into the already-ranked
score within a carousel.            organic list at fixed slot positions.
                                    e.g., slot 0 = ad, slots 1-4 = organic, repeat.

Lives in Layer 3.                   Runs in parallel with post-processing (Layer 4).
UBP Phase 1-2 scope.                UBP Phase 3 scope (needs calibrated value function).
```

These two systems are completely disconnected today — no shared scores, no shared models,
no shared config. Organic stores and ad stores cannot compete for slots because their scores
are on incomparable scales. UBP Phase 3 unifies them through calibration + the value function.

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

Two independent ranking problems, executed sequentially:

- **Vertical ranking**: which row (carousel) goes at which position?
- **Horizontal ranking**: within a row, which stores/items go in which slots?

That's the whole model. Everything else is implementation detail.

---

## 2. The Four-Layer Pipeline

```
REQUEST
   │
   ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 1: RETRIEVAL                                      │
│  Fetch candidate stores, items, carousels from upstream  │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 2: GROUPING                                       │
│  Bucket candidates into named carousels                  │
│  (Favorites, Taste, Reorder, NV, etc.)                   │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 3: HORIZONTAL RANKING          ← UBP scope        │
│  Within each carousel: rank stores/items by score        │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 4: VERTICAL RANKING            ← UBP scope        │
│  Across all carousels: rank carousels by score           │
│  Assign final sortOrder to each carousel                 │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
                  SERIALIZE → CLIENT
```

UBP governs layers 3 and 4. Retrieval and grouping are upstream inputs.

---

## 3. The ML Scoring Layer: Sibyl and Predictors

Every ranking score comes from **Sibyl** — DoorDash's internal ML prediction service. Feed-service sends it a batch of feature vectors (one per carousel or store) and receives a score per item.

A single Sibyl request can carry **multiple predictors at once**. For vertical ranking today, up to 7 predictors are called in one shot: a primary conversion predictor, a score multiplier, a programmatic boost predictor, UCB exploration, MAB, a vertical intent model, and VP profit predictors. Their outputs are assembled into one composite score.

An experiment typically changes **one thing** in this setup: a new primary model, a different intent model, enabling UCB, or adjusting a boost weight. It does not need to replace the entire scoring stack.

> See `current-system-deep-dive.md` for the full predictor roster and today's score assembly formula.

---

## 4. The Value Function

Every ranking decision answers the same question:

> **What is the expected value of showing this component to this user at this position?**

```
Expected Value = pImp × pAct × vAct

  pImp = P(user sees this component at position k)
         Decays with position. Row 0 > Row 3. Slot 0 > Slot 5.

  pAct = P(user takes action | they see it)
         Calibrated ML model output. Must be comparable across all content types.

  vAct = Value of that action
         GOV, FIV, ads revenue, NV trial value.
         Weights are configurable per experiment.
```

Today's scoring approximates this with a collection of multipliers and heuristics that encode business intent but aren't jointly optimizable or measurable. UBP makes the value function explicit and calibrated.

**Calibration is the critical path.** Without it, scores from different model families (organic UR, NV ranker, ads CTR) are on incomparable scales and cannot be ranked together. Everything else depends on calibration being right.

---

## 5. The Three Core Interfaces

Everything in UBP is built on three interfaces.

### Component — the unit being ranked

```kotlin
interface VerticalComponent {
    val id: String
    val type: ComponentType       // STORE_CAROUSEL, ITEM_CAROUSEL, NV_CAROUSEL, etc.
    var score: Double             // mutable — processors update this in place
    val metadata: MutableMap<String, Any>
    fun recordTrace(stepId: String, value: Any)
}
```

Today there are 9+ typed carousel classes with no shared interface. UBP collapses them to one.

### Processor — one step in the pipeline

```kotlin
interface VerticalProcessor {
    val type: String              // "MODEL_SCORING", "MULTIPLIER_BOOST", etc.

    suspend fun process(
        components: MutableList<VerticalComponent>,
        context: RankingContext,
        params: Map<String, Any>, // injected from config — not read internally
    )
}
```

`params` are injected at call time from the experiment config. Processors have no internal DV reads. This is what makes them reusable across experiments.

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

The engine has zero business logic. It reads the step list from config and dispatches to registered processors. A new processor type = one new class registered. A changed pipeline = changed JSON config. The engine itself never changes.

---

## 6. The Experiment Model: N × M

```
N experiments on horizontal ranking (Layer 3)
M experiments on vertical ranking   (Layer 4)

Each user is in exactly one horizontal variant
Each user is in exactly one vertical variant
A user can be in any (horizontal, vertical) combination

→ N × M valid experiment cells
→ Mutually exclusive within a layer, independent across layers
```

Implemented as independent DV keys — one per layer. This is already how DV works today. UBP formalizes it as a first-class platform property.

---

## 7. The Config Contract

There is one **platform-owned baseline** (`control`) that defines current production behavior. HP owns it. Every experiment inherits from it and declares only what it changes.

```
ubp/experiments/
  control.json              ← platform baseline, HP-owned, one per layer
  hp_vertical_fw_v4.json    ← "new model"
  hp_vertical_intent_v3.json
  nv_vertical_boost.json    ← NV team experiment
  ads_vertical_blend.json   ← Ads team experiment
```

An experiment file contains one or more **treatments**. A treatment either:
- **Overrides specific step params** (common) — declares only the delta from control
- **Declares a full pipeline** (rare) — when it needs a different step sequence

One DV key per experiment. One DV value selects which treatment runs. No code changes to run a new variant.

> See `experiment-config-contract.md` for the full contract schema and worked examples.

---

## 8. What "Unified" Means

**1. Unified Component Model**
All content types implement one interface. The engine ranks carousels without knowing whether they're store, item, NV, or ads.

**2. Unified Value Function**
All content types' scores express `pImp × pAct × vAct` on a calibrated, comparable scale. No content type gets a separate scoring system. Organic, ads, and merch compete on the same axis.

**3. Unified Experiment Platform**
Both layers are configured through the same JSON contract and DV convention. MLEs self-serve through one interface. Predictor selection, model versions, blending params, and pipeline shape all live in one place.

---

## 9. The Full Pipeline (Updated)

```
RETRIEVAL
    │
GROUPING (stores → carousels)
    │
ORGANIC HORIZONTAL RANKING  (Layer 3: Sibyl + business rules, per carousel)
    │
DECORATION + S1/S2/S3 RERANKING  (unobservable — runs after Iguazu fires)
    │
LAYOUT PROCESSING
    ├──────────────────────────────────────────────┐
    │                                              │
VERTICAL RANKING (Layer 4, post-processing)    ADS HORIZONTAL BLENDING
  Organic carousel order via UR + pinning        (HomepageAdsBlender, parallel)
                                                 Insert ad stores at fixed slots
    │                                              │
    └───────────────────────┬──────────────────────┘
                            │
                     SERIALIZATION → CLIENT
```

Two completely separate ranking pipelines merge at serialization.
They share nothing: no scores, no models, no config, no value function.

---

## 10. What the Current Code Gets Wrong

| Root Cause | Symptom | UBP Fix |
|---|---|---|
| No shared carousel interface | 9 typed classes, each ranks differently | `VerticalComponent` collapses all types |
| Predictor selection in scattered DV boolean checks | New predictor = new `if` in `P13nExperimentManager.kt` | Predictor config declared in experiment JSON |
| Model versions in a separate mapping JSON; intent model buried in blending config | Three files to touch for one experiment | Single experiment JSON: predictors + models + blending params |
| No config seam for step sequence | Adding a step requires a code change | Steps declared in config; engine dispatches from registry |
| Post-ranking fixups outside the pipeline | NV/PAD fixups run silently after the DAG, no traceability | All fixups declared as `FIXED_PINNING` steps in config |
| Manual Iguazu instrumentation per processor | Inconsistent, often missing | Engine auto-traces after every step |
| Organic/NV/ads scores on incomparable scales | Ads inserted after ranking; best ad may be buried | Calibration normalizes all content to one scale |

---

## 10. What Done Looks Like

```
An MLE running a new vertical ranking experiment:

  1. Trains a model → deploys to Sibyl
  2. Writes an experiment file with their treatment (just the delta from control)
  3. Adds one DV key
  4. Hot-deploys the JSON — no pod restart, no code review

The platform handles:
  ✓ Assembling the Sibyl request from config's predictor declarations
  ✓ Composing the multi-predictor score
  ✓ Applying calibration, intent, and boost steps per config
  ✓ Tracing every step automatically
  ✓ Traffic splitting via DV

The MLE never touches:
  ✗ Kotlin source code
  ✗ P13nExperimentManager
  ✗ dv_treatment_sibyl_model_mapping.json
  ✗ hp_vertical_blending_config.json
  ✗ Logging instrumentation
  ✗ Cross-team DV alignment
```

When that workflow works for both vertical and horizontal ranking, UBP is done.
