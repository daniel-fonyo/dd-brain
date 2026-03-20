# UBP: MLE Contract Interview Questions + Reference Examples

> Use this doc when meeting with the lead MLE to validate the contract design.
> The contract should be designed **for MLEs** — they are the primary consumers.
> All example JSONs reflect the actual UBP schema from `UnifiedExperimentConfig`.

---

## Interview Questions

### Opening — Frame the goal

1. **What is the most painful part of running a new vertical ranking experiment today?** (How long does it take from idea to first data?)
2. **Which config params do you change most often?** (Model, boost weights, features, or something else?)
3. **Have you ever wanted to run an experiment but couldn't because it required a code change?** What was it?

---

### What MLEs actually change

4. **When you launch a new model, what exactly do you want to swap?** Just the predictor name, or also the feature list, calibration, and intent layers simultaneously?
5. **How do you currently set boost weights for NV vs Rx carousels?** Do you ever want per-vertical-type weights or do carousel-level weights matter more?
6. **Do you care about diversity reranking as an MLE knob?** Or is that BE logic you don't want to touch?
7. **For value function weights (GOV, FIV, strategic) — do you want to tune these per experiment, or are they business policy set once by leadership?**
8. **What does "calibration" mean to you?** Is it a linear multiplier, a piecewise function, or something else?

---

### Inheritance / experiment structure

9. **Would you write a full config per experiment, or prefer to inherit from control and override only what changes?** (This determines whether we need the `"extends": "control"` field.)
10. **Do you run N × M experiments across ranking layers (e.g., a new model experiment × a new boost experiment)?** Or always one-at-a-time?
11. **Would you ever want two treatments to share a base that isn't `control`?** (e.g., `treatment_fw_v3 extends treatment_fw_v2`)

---

### Observability

12. **What traces do you look at after launching an experiment?** Score distributions, position changes, carousel type breakdown, or something else?
13. **Do you care about seeing per-step score snapshots** (after model scoring, after boost, after diversity)? Or just final sort order?
14. **When something goes wrong (wrong carousel ranked high, expected carousel missing), how do you debug it today?**

---

### Contract validation

15. **Does this contract capture everything you'd need to run your next experiment?** What's missing?
16. **Are there params you never want to configure (e.g., pImp decay rate, RPC retry counts)?** Those should be BE-only and not appear in the contract.
17. **What is your rollback story if an experiment goes badly?** Does "just change the DV assignment back to control" cover it, or do you need something faster?

---

## Contract Reference Examples

All examples use the canonical `UnifiedExperimentConfig` schema.
Engine step order is **always hardcoded**: `MODEL_SCORING → BOOST_AND_RANK`.

---

### Schema reference (field dictionary)

```
UnifiedExperimentConfig
├── p_act                          Probability of Action (MLE-owned)
│   ├── predictor                  Sibyl predictor name
│   ├── model                      Model artifact name
│   └── features[]                 Feature keys sent to predictor
├── v_act                          Value of Action (business outcome weights)
│   ├── gov                        Gross Order Value weight
│   ├── fiv                        First Impression Value weight (NV lifestage)
│   └── strategic                  Strategic business priority weight
├── boost_and_rank                 BOOST_AND_RANK step params
│   ├── boost_by_position_enabled  Whether position boosting is active
│   ├── deal_carousel_score_multiplier  Multiplier for deal carousels
│   └── boost_by_position_allow_list   Carousel IDs eligible for position boost
└── extends                        Inherit from named config, merge overrides on top
```

**Ownership summary:**
| Variable | Formula | Owner |
|---|---|---|
| `p_act.predictor` + `p_act.model` | pAct | MLE |
| `v_act.*` | vAct | MLE (or business policy) |
| `boost_and_rank.*` | BOOST_AND_RANK params | MLE |
| `pImp` (position decay rate) | pImp | BE only — not in contract |
| Sibyl RPC, retries, timeouts | — | BE only |
| Trace schema | — | BE only, auto-emitted |

---

### Example 1 — Control baseline (today's behavior, declared)

This is the `control.json` that exactly mirrors the current production pipeline.
It is the inheritance base for all experiments.

```json
{
  "control": {
    "p_act": {
      "predictor": "feed_ranking_v1",
      "model": "store_ranker_v1",
      "features": ["STORE_ID", "CONSUMER_ID", "STORE_ETA", "DAY_OF_WEEK", "IS_DASHPASS"]
    },
    "v_act": {
      "gov": 1.0,
      "fiv": 0.0,
      "strategic": 0.0
    },
    "boost_and_rank": {
      "boost_by_position_enabled": false,
      "deal_carousel_score_multiplier": 1.0,
      "boost_by_position_allow_list": []
    }
  }
}
```

**What this declares:**
- Standard feed_ranking_v1 Sibyl predictor, store_ranker_v1 model
- Pure GOV optimization (no FIV or strategic weight)
- No boost multipliers (all 1.0× — no effect)
- Diversity reranking disabled
- No fixed pins (post-ranking fixups currently happen in code; this is the target state)

---

### Example 2 — New model experiment (swap predictor + features only)

Most common MLE workflow: replace the model, keep everything else.

```json
{
  "treatment_intent_v3": {
    "extends": "control",
    "p_act": {
      "predictor": "feed_ranking_fw_v1",
      "model": "vertical_intent_v3",
      "features": [
        "STORE_ID",
        "CONSUMER_ID",
        "DAY_OF_WEEK",
        "IS_DASHPASS",
        "NV_LIFESTAGE",
        "SCENE_ID",
        "CONTAINER_CODE"
      ]
    }
  }
}
```

**What changes:** predictor, model artifact, and feature set.
**What's inherited from control:** v_act, boost_weights, diversity, pins.
**Why `extends`:** MLE writes only what changes. No need to copy-paste control params.

---

### Example 3 — NV strategic boost (tune value weights + boost multipliers)

Rebalance GOV vs strategic value while boosting NV carousel visibility.

```json
{
  "treatment_nv_boost_v2": {
    "extends": "control",
    "v_act": {
      "gov": 0.8,
      "fiv": 0.1,
      "strategic": 0.1
    },
    "boost_and_rank": {
      "boost_by_position_enabled": true,
      "deal_carousel_score_multiplier": 2.5,
      "boost_by_position_allow_list": ["nv_carousel"]
    }
  }
}
```

**What changes:** value function rebalanced toward NV/strategic; boost-and-rank params enable position boosting.
**What's inherited:** model, features.

**Key insight from audit trail:** boost_weights lookup priority is `carousel_id` > `business_vertical_id` > `vertical_id` > default.
A carousel with `vertical_id=100322` (NV) but `business_vertical_id=169` (matched a different rule) will get the wrong multiplier.
Under UBP, this is explicit in the JSON — no hidden lookup priority bugs.

---

### Example 4 — Intent model + diversity reranking (combined)

New model that emphasizes vertical intent, plus diversity reranking to prevent NV monopoly on page.

```json
{
  "treatment_intent_diversity": {
    "extends": "control",
    "p_act": {
      "predictor": "feed_ranking_fw_v1",
      "model": "vertical_intent_v3",
      "features": [
        "STORE_ID",
        "CONSUMER_ID",
        "DAY_OF_WEEK",
        "IS_DASHPASS",
        "NV_LIFESTAGE"
      ]
    },
    "v_act": {
      "gov": 0.7,
      "fiv": 0.2,
      "strategic": 0.1
    },
    "boost_and_rank": {
      "boost_by_position_enabled": false,
      "deal_carousel_score_multiplier": 1.0,
      "boost_by_position_allow_list": []
    }
  }
}
```

**What changes:** model, features, value weights.
**What's inherited:** boost_and_rank defaults.

---

### Example 5 — Boost-and-rank with position boosting enabled

Position boosting and pinning are now part of the `BOOST_AND_RANK` step.

```json
{
  "treatment_boost_enabled": {
    "extends": "control",
    "boost_and_rank": {
      "boost_by_position_enabled": true,
      "deal_carousel_score_multiplier": 1.5,
      "boost_by_position_allow_list": ["dt:some_carousel", "nv_carousel"]
    }
  }
}
```

**Note:** Business fixups (NV post-checkout pinning, PAD position=3) remain outside the pipeline
in `NonRankableHomepageOrderingUtil`. Only MLE-configurable boost params are inside `BOOST_AND_RANK`.

---

### Example 6 — N × M cross-layer experiment

Vertical experiments and horizontal experiments are independent axes.
A user can be in `treatment_intent_v3` (vertical) AND `treatment_fw_v2` (horizontal) simultaneously.

```
DV "ubp_hp_vertical_v3"   → "treatment_intent_v3"   (Layer 4 assignment)
DV "ubp_hp_horizontal_v2" → "treatment_fw_v2"        (Layer 3 assignment)
```

Vertical config (`ubp/experiments/vertical_v3.json`):
```json
{
	...,
  "vp_experiment": {
    "extends": "control",
    "p_act": {
      "predictor": "feed_ranking_fw_v1",
      "model": "vertical_intent_v3",
      "weights": {
		    cost: 0.3,
		    profit: 0.1
      }
    }
  }
}
```

Horizontal config (`ubp/experiments/horizontal_v2.json`):
```json
{
  "treatment_fw_v2": {
    "extends": "control",
    "p_act": {
      "predictor": "feed_ranking_fw_v1",
      "model": "store_ranker_fw_v2",
      "features": ["STORE_ID", "CONSUMER_ID", "STORE_ETA", "SCENE_ID", "CONTAINER_CODE", "IS_DASHPASS"]
    }
  }
}
```

**Result:** N × M valid experiment cells. No code coupling between layers.

---

## What the Audit Trail Revealed (Reality Check)

From the real log dump (`consumer=757606047`, `trace_id=f61f3cdb...`):

| What we expected | What we found |
|---|---|
| Calibration multipliers vary | All = 1.0 (control config has no calibration) |
| Intent multipliers vary | All = 1.0 (no intent model in control) |
| Boost weights are the lever | Confirmed: only active ranking signal |
| All 9 carousel types flow through BlendingUtil | Only StoreCarousel + ItemCarousel — 7 types silently excluded via `else → null` |
| DealCarousel competes on same scale as StoreCarousel | False — excluded from blending, competes on raw Sibyl score (incomparable scale) |
| NV boost = 5.0× for carousel `07e18b95` | Got 1.25× — `business_vertical_id=169` lookup hit before `vertical_id=100322` |

**The UBP contract makes every one of these explicit and auditable.**

---

## DV Wiring Convention

| Config layer | DV key pattern | JSON path |
|---|---|---|
| Vertical | `ubp_hp_vertical_{experiment_id}` | `ubp/experiments/{experiment_id}.json` |
| Horizontal | `ubp_hp_horizontal_{experiment_id}` | `ubp/experiments/{experiment_id}.json` |

Engine reads DV assignment → looks up JSON key by assignment value → resolves `extends` chain → injects params into processors.

```

```
- VP
	- dasher cost mode and comish fee models
-  Exploration ranker 
	- Runs after UR
- How to validate values from contract?
	- Need to check output score makes sense? Can we enforce this?
- Per segment logic per DV thats hardcoded
	- How does runtime config handle this?
