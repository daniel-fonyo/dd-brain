# UBP Experiment Config Contract — Examples

> Companion to `northstar.md`. Shows what experiment JSON files actually look like.
> Start here when writing a new experiment.

---

## The Baseline: `control.json`

The control is the canonical definition of production behavior. HP owns it. All experiments inherit from it.

```json
// ubp/experiments/control.json
{
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
```

---

## Experiment File Structure

An experiment file lives at `ubp/experiments/{experiment_id}.json`. It contains one or more named treatments. Each treatment either:

- **Overrides step params** (`step_params`) — the common case; inherits the full pipeline from control
- **Declares a full pipeline** (`vertical_pipeline`) — when the step sequence itself changes

The DV key `ubp_{experiment_id}` maps to a treatment name in this file. If no match is found, the engine falls back to control.

---

## Case 1: New Primary Model

The most common experiment. Same pipeline, different model version.

```json
// ubp/experiments/hp_vertical_fw_v4.json
{
  "treatment_fw_v4": {
    "extends": "control",
    "step_params": {
      "score": { "primary_model": "store_ranker_fw_v4" }
    }
  }
}
```

DV: `ubp_hp_vertical_fw_v4` → `treatment_fw_v4`

---

## Case 2: Adjust a Boost Weight

Testing whether a different NV boost improves metrics.

```json
// ubp/experiments/nv_vertical_boost.json
{
  "treatment_nv_1_5x": {
    "extends": "control",
    "step_params": {
      "blend": { "vertical_boost_weights": { "1": 1.0, "10": 1.5 } }
    }
  },
  "treatment_nv_2x": {
    "extends": "control",
    "step_params": {
      "blend": { "vertical_boost_weights": { "1": 1.0, "10": 2.0 } }
    }
  }
}
```

Multiple treatments in one file. DV assigns users to one treatment name.

---

## Case 3: Enable UCB Exploration

Testing exploration bonus for under-exposed stores.

```json
// ubp/experiments/hp_vertical_ucb.json
{
  "treatment_ucb_v2": {
    "extends": "control",
    "step_params": {
      "score": {
        "ucb_enabled": true,
        "ucb_predictor": "ucb_uncertainty",
        "ucb_model": "ucb_v2"
      }
    }
  }
}
```

---

## Case 4: New Intent Model

Testing a new intent model that requires its own calibration table and intent lookup.

```json
// ubp/experiments/hp_vertical_intent_v3.json
{
  "treatment_intent_v3": {
    "extends": "control",
    "step_params": {
      "score": {
        "intent_model": "intent_v3_model_id"
      },
      "calibrate": {
        "calibration_table": "intent_v3_calibration"
      },
      "blend": {
        "intent_lookup_table": "intent_v3_lookup"
      }
    }
  }
}
```

Three step overrides, but still a param-only experiment — no change to pipeline shape.

---

## Case 5: New Primary Model + Intent Together

Two variables changed in one treatment.

```json
// ubp/experiments/hp_vertical_fw_v4_intent_v3.json
{
  "treatment_fw_v4_intent_v3": {
    "extends": "control",
    "step_params": {
      "score": {
        "primary_model": "store_ranker_fw_v4",
        "intent_model": "intent_v3_model_id",
        "ucb_enabled": true,
        "ucb_model": "ucb_v2"
      },
      "calibrate": { "calibration_table": "intent_v3_calibration" },
      "blend":     { "intent_lookup_table": "intent_v3_lookup" }
    }
  }
}
```

---

## Case 6: New Pipeline Step (Structural Change)

Testing diversity reranking, which does not exist in control. The step sequence changes, so the full pipeline must be declared.

```json
// ubp/experiments/hp_vertical_diversity.json
{
  "treatment_diversity_v1": {
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
          "id": "diversity",
          "type": "DIVERSITY_RERANK",
          "params": {
            "weight": 0.4,
            "rx_nrx_balance": 0.6
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

If `DIVERSITY_RERANK` is a new processor type not yet registered, a BE engineer adds it to the registry once. After that, any experiment can use it via config.

---

## Merge Semantics

For `step_params` overrides, the engine:

1. Starts with the full control pipeline
2. For each step id in `step_params`, shallow-merges the params into that step's params (treatment value wins on collision)
3. Missing step ids in `step_params` are left unchanged

**Shallow merge** means the entire param value is replaced, not merged recursively. If `vertical_boost_weights` in control is `{ "1": 1.0, "10": 1.2 }` and the treatment sets `{ "1": 1.0, "10": 1.5 }`, the full map is replaced. You must re-declare all keys you want to keep.

If a treatment declares `vertical_pipeline` directly, it takes full precedence — no merge with control.

---

## Decision Guide: Which Case Are You?

| What you're changing | Use |
|---|---|
| New primary model version | Case 1: one line in `step_params.score` |
| Adjust a boost weight | Case 2: one entry in `step_params.blend` |
| Enable/disable UCB or MAB | Case 3: toggle + model in `step_params.score` |
| New intent model | Case 4: intent model + calibration + lookup across 3 step_params |
| Multiple things at once | Case 5: combine step_params entries |
| New step type (new pipeline stage) | Case 6: declare full `vertical_pipeline`; may need BE to register processor |
