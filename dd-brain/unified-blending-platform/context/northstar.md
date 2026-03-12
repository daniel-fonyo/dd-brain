# Unified Blending Platform — Engineering First Principles

> This is the northstar. When in doubt, return here.

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

## 2. The Four-Layer Pipeline

```
REQUEST
   │
   ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 1: RETRIEVAL                                      │
│  Fetch candidate stores, items, carousels from upstream  │
│  Output: raw pool of ~200-1000 candidates                │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 2: GROUPING                                       │
│  Bucket candidates into named carousels                  │
│  (Favorites, Taste, Reorder, NV, etc.)                   │
│  Output: ~20 carousels, each with ~10-50 candidates      │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 3: HORIZONTAL RANKING                             │
│  Within each carousel: rank stores/items by score        │
│  Output: same carousels, stores now in ranked order      │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  LAYER 4: VERTICAL RANKING                               │
│  Across all carousels: rank carousels by score           │
│  Assign final sortOrder to each carousel                 │
│  Output: final sorted carousel list                      │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
                  SERIALIZE → CLIENT
```

UBP is the platform that governs **layers 3 and 4**. Retrieval and grouping are upstream inputs.

---

## 3. Sibyl and Predictors: The ML Scoring Layer

Before discussing the value function, you need to understand how ML scores are actually obtained. Every ranking decision goes through **Sibyl** — DoorDash's internal ML prediction service.

### What Sibyl Does

Feed-service sends Sibyl a batch of **feature vectors** (one per carousel or store being ranked) and receives back a **score per item per predictor**. A single request can ask multiple predictors at once.

```
Feed-Service ──────────────────────────────────────────► Sibyl
             PredictionRequest {
               predictors: [mainPredictor, multiplierPredictor, ucbPredictor, ...]
               featureSets: [carousel_0_features, carousel_1_features, ...]
               useCase: "home_page_vertical_ranking"
             }

Sibyl ─────────────────────────────────────────────────► Feed-Service
      PredictionResponse {
        per predictor, per featureSet: { score: Double }
      }
```

### The Predictor Roster

A **predictor** is a named slot in Sibyl that serves a specific model family. A **model override** is the specific trained version within that slot. Today's vertical ranking uses up to 7 predictors per request:

| Predictor | Role | Combined How |
|---|---|---|
| **Main (FW)** | Core conversion probability — the primary `pAct` signal | Multiplicative base |
| **Multiplier** | Per-store scaling factor (store ranker boost) | `× multiplierScore` |
| **Programmatic Boost** | Campaign/business-rule boost | `× progBoostScore` |
| **UCB Uncertainty** | Exploration bonus for under-explored stores | `+ ucbScore` |
| **MAB** | Multi-armed bandit bonus | `+ mabScore` |
| **Vertical Intent** | User intent signal (RX vs. NV preference) | Applied in blending stage |
| **VP Predictors** | `pDasherCost` + `pCommission` for profit-based ranking | Replaces base formula |

The **composite Sibyl score** from Stage 1 (before blending):

```
sibylScore = (mainScore × multiplierScore × progBoostScore)
           + ucbScore
           + mabScore
```

This score is then further adjusted in the blending stage (see Section 5).

### How an Experiment Selects Predictors and Models

An experiment selects its ML model through two independent mechanisms:

**1. Predictor switch (via DV boolean):**
Whether to use the FW predictor, enable UCB, enable MAB, etc. — each controlled by its own DV flag in `P13nExperimentManager`. These are scattered boolean checks today. UBP declares them as `predictor_config` in the experiment JSON.

**2. Model version (via DV → JSON lookup):**
The specific trained model artifact for each predictor is resolved via a runtime JSON map:

```
DV value: "treatment_fw_v4"
     │
     ▼
dv_treatment_sibyl_model_mapping.json
{
  "FEED_RANKING_FW": { "treatment_fw_v4": "store_ranker_fw_v4_model_id" },
  "MULTIPLIER":      { "treatment_fw_v4": "multiplier_v4_model_id" }
}
     │
     ▼
Sibyl called with modelOverrideId = "store_ranker_fw_v4_model_id"
```

**The problem today:** these two mechanisms are separate files, separate DV keys, and separate code paths. Adding a new predictor to an experiment means touching `P13nExperimentManager.kt` (boolean check), `dv_treatment_sibyl_model_mapping.json` (model IDs), and `hp_vertical_blending_config.json` (blending params) — plus a separate entry for the intent model name, which lives *inside* the blending config. UBP collapses all of this into one experiment JSON.

---

## 4. The Value Function

Every ranking decision — whether ranking a store within a carousel or a carousel on the page — answers the same question:

> **What is the expected value of showing this component to this user at this position?**

```
Expected Value = pImp × pAct × vAct

  pImp = P(user sees this component at position k)
         Decays with position. Row 0 > Row 3. Slot 0 > Slot 5.

  pAct = P(user takes action | they see it)
         The calibrated ML model output. Predicted CTR, CVR, order probability.
         Must be on a comparable scale across all content types.

  vAct = Value of that action (in business terms)
         GOV (gross order value), FIV (first-impression value),
         ads revenue, strategic value (NV trial, retention).
         Weights are configurable per experiment.
```

**What exists today vs. what this requires:**

| Term | Today's approximation | What's missing |
|---|---|---|
| `pAct` | Main Sibyl predictor output (conversion probability) | Calibration to comparable scale across content types |
| `vAct` | VP experiment: `pConv × pDasherCost × pCommission × α × β`; otherwise: scattered multipliers and boost weights | A unified, declared weight vector. Organic/NV/ads/merch on the same scale |
| `pImp` | **Not modeled.** Position assigned after scoring, not used to influence it | Position-decay model; joint optimization of score and slot assignment |

The current scoring is not wrong — it is a pragmatic approximation. But it is an approximation: a collection of multiplicative heuristics assembled over time that encode business intent without being measurable or jointly optimizable. UBP's job is to make this an explicit, calibrated, measurable value function.

**Why calibration is the critical path:** `pAct` outputs from different models (organic UR, NV ranker, ads CTR model) are not on comparable scales. Without calibration, you cannot rank organic carousels against NV carousels against ads on the same `pAct × vAct` axis. Every other UBP capability depends on calibration being correct first.

---

## 5. How Blending Works Today (and What UBP Replaces)

After the Sibyl composite score is assembled, a second scoring stage applies three multiplicative adjustments in `BlendingUtil`:

```
finalScore = sibylScore
           × calibration(verticalId, scoreBucket)     ← piecewise range lookup
           × intentPredictionScore                    ← from intent Sibyl model
           × intentLookupMultiplier(contextFeatures)  ← feature lookup table
           × verticalBoostWeight(verticalId)          ← static per-vertical multiplier
           + diversityBonus                           ← if diversity reranking enabled
```

- **Calibration** normalizes scores across verticals so they're roughly comparable — a manual piecewise table, updated offline
- **Intent** combines a Sibyl intent model output with a feature lookup table encoding user cohort × vertical affinity (e.g., dashpass users boosted toward NV)
- **Vertical boost weight** is a static hand-tuned multiplier per vertical — the most nakedly heuristic piece

In UBP, these all become declared `MULTIPLIER_BOOST` and `CALIBRATION` steps in the pipeline config, with params injected from the experiment JSON. The logic is unchanged; the wiring and observability are what change.

---

## 6. The Three Core Interfaces

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
    val type: String                 // "MODEL_SCORING", "MULTIPLIER_BOOST", etc.

    suspend fun process(
        components: MutableList<VerticalComponent>,
        context: RankingContext,
        params: Map<String, Any>,   // injected from config, not read internally
    )
}
```

The critical property: `params` are **injected at call time**, not pulled from DV keys inside the processor. This is what makes processors reusable across experiments.

**The `MODEL_SCORING` processor in detail:** this processor is not a thin wrapper around one Sibyl call. It is responsible for assembling the full `RegressionContext` — which predictors to use, which model versions, whether to enable UCB/MAB/boost — and assembling the composite score from all predictor outputs. The params it receives from config declare the full predictor configuration:

```json
{
  "type": "MODEL_SCORING",
  "params": {
    "primary_predictor": "feed_ranking_fw",
    "primary_model": "store_ranker_fw_v4",
    "multiplier_predictor": "universal_ranker_multiplier",
    "multiplier_model": "multiplier_v4",
    "ucb_enabled": true,
    "ucb_predictor": "ucb_uncertainty",
    "ucb_model": "ucb_v2",
    "mab_enabled": false,
    "intent_predictor": "vertical_intent",
    "intent_model": "intent_v3"
  }
}
```

This is the config contract that replaces today's scattered DV boolean checks + model mapping JSON + blending config intent model field.

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

## 7. The Experiment Model: N × M

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

## 8. The Config Contract

One JSON file per experiment group. One DV controls which variant runs. No code changes to run a new experiment variant.

### Two experiment shapes

Most experiments only change one variable — a new model, a different boost weight, enabling UCB. Forcing every experiment to re-declare the entire pipeline is the wrong design: it creates copy-paste drift, makes diffs unreadable, and breaks when control changes.

UBP uses an **inheritance + override** pattern:

- Every experiment file has a `"control"` variant that declares the full pipeline
- All other variants use `"extends": "control"` and declare only **what differs** via `"step_params"`
- The engine deep-merges: experiment step_params override the base params at the leaf level
- A variant can also declare a full `"vertical_pipeline"` if it needs a structural change (different step sequence)

### Control: the full pipeline (declared once)

```json
// ubp/experiments/hp_vertical_blend.json
{
  "control": {
    "vertical_pipeline": {
      "steps": [
        {
          "id": "score",
          "type": "MODEL_SCORING",
          "params": {
            "primary_predictor": "feed_ranking_fw",
            "primary_model": "store_ranker_fw_v3",
            "multiplier_predictor": "universal_ranker_multiplier",
            "multiplier_model": "multiplier_v3",
            "ucb_enabled": false,
            "mab_enabled": false,
            "intent_predictor": "vertical_intent",
            "intent_model": ""
          }
        },
        {
          "id": "calibrate",
          "type": "CALIBRATION",
          "params": { "calibration_table": "default_piecewise_v1" }
        },
        {
          "id": "blend",
          "type": "MULTIPLIER_BOOST",
          "params": {
            "vertical_boost_weights": { "1": 1.0, "10": 1.2 },
            "intent_lookup_table": "default_intent_v1"
          }
        },
        {
          "id": "pin",
          "type": "FIXED_PINNING",
          "params": { "pinning_rules": "default_pinned_order" }
        }
      ]
    }
  }
}
```

### Param-only experiments: declare only the delta

The common case. Same pipeline shape as control; one or a few params differ.

```json
// "I trained a new main model. Everything else is identical."
"treatment_fw_v4": {
  "extends": "control",
  "step_params": {
    "score": { "primary_model": "store_ranker_fw_v4" }
  }
},

// "I want to test a higher NV boost weight."
"treatment_nv_boost": {
  "extends": "control",
  "step_params": {
    "blend": { "vertical_boost_weights": { "1": 1.0, "10": 1.5 } }
  }
},

// "I want to test enabling UCB exploration."
"treatment_ucb_v2": {
  "extends": "control",
  "step_params": {
    "score": { "ucb_enabled": true, "ucb_model": "ucb_v2" }
  }
},

// "I want to test a new intent model with its own calibration."
"treatment_intent_v3": {
  "extends": "control",
  "step_params": {
    "score": {
      "intent_model": "intent_v3",
      "ucb_enabled": true,
      "ucb_model": "ucb_v2"
    },
    "calibrate": { "calibration_table": "intent_v3_calibration" },
    "blend":     { "intent_lookup_table": "intent_v3_lookup" }
  }
}
```

Each treatment declares only what it changes. The engine resolves the full config at request time by merging `step_params` over the base.

### Structural experiments: new step sequence (rare)

When an experiment adds or removes a step (not just changes a param), it declares the full pipeline. This makes the structural change explicit and intentional:

```json
// "I want to test diversity reranking, which doesn't exist in control."
"treatment_diversity_v1": {
  "extends": "control",
  "vertical_pipeline": {
    "steps": [
      { "id": "score",     "type": "MODEL_SCORING",    "params": { ... } },
      { "id": "calibrate", "type": "CALIBRATION",      "params": { ... } },
      { "id": "blend",     "type": "MULTIPLIER_BOOST", "params": { ... } },
      { "id": "diversity", "type": "DIVERSITY_RERANK",  "params": { "weight": 0.4 } },
      { "id": "pin",       "type": "FIXED_PINNING",    "params": { ... } }
    ]
  }
}
```

### Merge semantics

The engine applies these rules at config resolution time:

1. Start with the base variant's full `vertical_pipeline.steps`
2. For each step in `step_params`, find the matching step by `id` and shallow-merge the params (experiment value wins on any key collision)
3. If the treatment declares `vertical_pipeline` directly, it takes full precedence — no merge
4. If no base is found, fail fast with an error (no silent fallback to partial config)

**Shallow merge at the step level:** if `blend` params in control are `{ "vertical_boost_weights": { "1": 1.0, "10": 1.2 }, "intent_lookup_table": "default_intent_v1" }` and the treatment overrides `"vertical_boost_weights": { "1": 1.0, "10": 1.5 }`, the full `vertical_boost_weights` map is replaced, not individual keys within it. This prevents partial-map confusion.

### Properties of this contract

- **Experiments declare what they own** — the diff is explicit and reviewable; a model swap is 1 line
- **Control is the single source of truth** for the pipeline shape — when control evolves, param-only experiments automatically inherit the change
- **Structural changes require full declaration** — this is intentional friction; adding a step is a bigger decision than changing a weight
- **All predictor config is in one place** — model versions, predictor toggles, blending params co-located; no separate DV boolean checks, no separate model mapping JSON
- **MLE surface:** for a param-only experiment, edit 1–3 lines of JSON and add one DV key
- **BE surface:** register a new `Processor` class when a new step type is needed; engine handles the rest

---

## 9. What "Unified" Means

"Unified" has three concrete meanings:

**1. Unified Component Model**
Every content type (store carousel, item carousel, NV carousel, ads carousel) implements the same `Component` interface. The ranking engine doesn't know or care what type of content it's ranking. There is no `if (component is StoreCarousel)` in the engine.

**2. Unified Value Function**
Every content type's score is expressed as `pImp × pAct × vAct` on a comparable, calibrated scale. Organic stores, ads, merch, and NV placements compete on the same scale. No content type gets a separate scoring system or a hardcoded multiplier that can't be measured. This requires calibration as a prerequisite.

**3. Unified Experiment Platform**
Both horizontal and vertical ranking are configured through the same JSON contract, the same DV wiring convention, and the same tracing infrastructure. MLEs self-serve both layers through the same interface. No layer requires a different workflow. Predictor selection, model versioning, blending params, and pipeline shape are all declared in one place.

---

## 10. What the Current Code Gets Wrong

These are the root causes that UBP addresses:

| Root Cause | Symptom | UBP Fix |
|---|---|---|
| Each carousel type owns its own ranking logic | FavoritesCarousel, DVCarousel, TasteCarousel each do ranking differently, no shared interface | `Component` interface collapses all types; `Processor` registry handles step logic |
| Predictor selection spread across boolean DV checks | Adding a new predictor requires a new `if` in `P13nExperimentManager.kt` per experiment | Predictor config declared in experiment JSON; `MODEL_SCORING` processor reads it |
| Model versions in separate mapping JSON | Model IDs and blending params live in different files; intent model name buried in blending config | Single experiment JSON contains all predictor names + model versions + blending params |
| Step params live in 10+ scattered places | Changing one experiment touches 5 DVs + 3 JSON files + hardcoded constants | Single experiment JSON per treatment; params injected at call time |
| No config seam for step sequence | Adding/removing a step requires code change | Steps declared in config; engine composes from registry at request time |
| Post-ranking fixups are outside the pipeline | `updateSortOrderOfNVCarousel()` runs after the DAG with no traceability | All fixups become declared `FIXED_PINNING` steps in the config |
| Manual per-step Iguazu instrumentation | Each processor manually calls IguazuEventUtil — inconsistent, often missing | Engine auto-calls `recordTrace()` after every step; standard schema |
| Organic/ads/NV scores are incomparable | Ads inserted AFTER organic ranking; no calibration across content types | All content declares `pAct` + `vAct`; calibration service normalizes to comparable scale |
| `pImp` not modeled | Position assigned after scoring; score doesn't account for impression probability | Future: position-decay model jointly optimizes score and slot assignment |

---

## 11. The Shared Baseline: One Control to Rule Them All

The control is not per-experiment. It is a **single, platform-owned baseline** that represents current production behavior. It lives at:

```
ubp/experiments/control.json
```

HP owns it. It is the canonical definition of what the pipeline does for users in no experiment. When a winning treatment is shipped to 100%, the control is updated to match — that's how the platform moves forward.

Every experiment inherits from this baseline. MLEs never write a "control" — they write treatments.

```
ubp/experiments/
  control.json                  ← platform baseline, HP-owned
  hp_vertical_fw_v4.json        ← "I trained a new model"
  hp_vertical_intent_v3.json    ← "I want to test an intent model"
  nv_vertical_boost.json        ← "NV team wants to test a higher NV weight"
  ads_vertical_blend.json       ← "Ads team wants to test inserting ads in vertical"
```

Any part of the control pipeline is fair game for any team to experiment on. The MLE owns their treatment; the platform owns the baseline.

---

## 12. The Northstar: What Done Looks Like

For a **param-only experiment** (the common case — new model, weight change, toggle):

```
An MLE who wants to test a new vertical ranking model:

  1. Trains a new model → deployed to Sibyl under a new model ID
  2. Writes 3 lines of JSON in a new experiment file:
       {
         "treatment_fw_v4": {
           "extends": "control",
           "step_params": { "score": { "primary_model": "store_ranker_fw_v4" } }
         }
       }
  3. Adds one DV key: UBP_HP_VERTICAL_FW_V4("ubp_hp_vertical_fw_v4")
  4. Deploys: runtime JSON hot-reload, no pod restart

That's it.
```

For a **structural experiment** (new step, different pipeline):

```
An MLE who wants to test diversity reranking (a new step):

  1. Writes a full pipeline declaration in a new experiment file
     (copies control, adds the DIVERSITY_RERANK step at the right position)
  2. Adds one DV key
  3. Deploys

A new processor may need to be registered by a BE engineer —
this is the only code change. Everything else is config.
```

**What the BE engine handles automatically, for both cases:**
- Assembling the RegressionContext from config's predictor declarations
- Calling Sibyl with all configured predictors in one request
- Composing the multi-predictor score (main × multiplier × boost + ucb + mab)
- Applying calibration, intent, and boost multipliers per config
- Tracing after every step (automatic, standard schema)
- Traffic splitting (via DV infrastructure)

**What the MLE never touches:**
- Java/Kotlin source code (for param-only experiments, ever)
- `P13nExperimentManager` boolean checks
- `dv_treatment_sibyl_model_mapping.json`
- `hp_vertical_blending_config.json` (separate file, separate DV today — gone in UBP)
- Manual logging instrumentation
- Cross-team alignment for DV wiring

The day that workflow is possible for both vertical and horizontal ranking, UBP is done.
