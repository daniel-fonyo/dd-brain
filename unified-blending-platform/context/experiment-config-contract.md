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
        "id": "boost_and_rank",
        "type": "BOOST_AND_RANK",
        "params": {
          "boost_by_position_enabled": false,
          "deal_carousel_score_multiplier": 1.0,
          "boost_by_position_allow_list": []
        }
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

## Case 2: Adjust Boost-and-Rank Params

Testing whether different boost settings improve metrics.

```json
// ubp/experiments/nv_vertical_boost.json
{
  "treatment_boost_enabled": {
    "extends": "control",
    "step_params": {
      "boost_and_rank": {
        "boost_by_position_enabled": true,
        "deal_carousel_score_multiplier": 1.5,
        "boost_by_position_allow_list": ["dt:some_carousel"]
      }
    }
  },
  "treatment_boost_2x": {
    "extends": "control",
    "step_params": {
      "boost_and_rank": {
        "boost_by_position_enabled": true,
        "deal_carousel_score_multiplier": 2.0,
        "boost_by_position_allow_list": ["dt:some_carousel"]
      }
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

Testing a new intent model. Under the new 2-step decomposition, model scoring params
(including intent) are part of the `MODEL_SCORING` step.

```json
// ubp/experiments/hp_vertical_intent_v3.json
{
  "treatment_intent_v3": {
    "extends": "control",
    "step_params": {
      "score": {
        "model_name": "intent_v3_model_id"
      }
    }
  }
}
```

Single step override — no change to pipeline shape.

---

## Case 5: New Primary Model + Intent Together

Two variables changed in one treatment. Both are MODEL_SCORING params under the new decomposition.

```json
// ubp/experiments/hp_vertical_fw_v4_intent_v3.json
{
  "treatment_fw_v4_intent_v3": {
    "extends": "control",
    "step_params": {
      "score": {
        "model_name": "store_ranker_fw_v4"
      }
    }
  }
}
```

---

## Case 6: New Pipeline Step (Structural Change)

Testing a new step type (e.g. MERCH_PLACEMENT) that does not exist in control. The step
sequence changes, so the full pipeline must be declared.

```json
// ubp/experiments/hp_vertical_merch.json
{
  "treatment_merch_v1": {
    "vertical_pipeline": {
      "steps": [
        {
          "id": "score",
          "type": "MODEL_SCORING",
          "params": { "predictor_ref": "p_act" }
        },
        {
          "id": "boost_and_rank",
          "type": "BOOST_AND_RANK",
          "params": {
            "boost_by_position_enabled": false,
            "deal_carousel_score_multiplier": 1.0,
            "boost_by_position_allow_list": []
          }
        },
        {
          "id": "merch",
          "type": "MERCH_PLACEMENT",
          "params": { "campaign_boost": 1.3 }
        }
      ]
    }
  }
}
```

If `MERCH_PLACEMENT` is a new processor type not yet registered, a BE engineer adds it to the registry once. After that, any experiment can use it via config.

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
| Adjust boost-and-rank params | Case 2: override `step_params.boost_and_rank` |
| Enable/disable UCB or MAB | Case 3: toggle + model in `step_params.score` |
| New intent model | Case 4: override model in `step_params.score` |
| Multiple things at once | Case 5: combine step_params entries |
| New step type (new pipeline stage) | Case 6: declare full `vertical_pipeline`; may need BE to register processor |
