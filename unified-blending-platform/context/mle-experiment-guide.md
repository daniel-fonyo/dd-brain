# MLE Experiment Guide — UBP Vertical Ranking

Reference for MLEs authoring experiment configs for the Unified Blending Platform.
See `spec/rfc.md` for the full contracts and architecture.

---

## Quick Decision Guide

| What you're changing | Mode | Example below |
|---|---|---|
| New model version | Mode 1 — override `p_act` | Case 1 |
| Adjust a boost weight | Mode 1 — override `step_params.blend` | Case 2 |
| New intent model + calibration | Mode 1 — override `p_act` + `step_params.blend` | Case 3 |
| Enable diversity reranking | Mode 1 — override `step_params.diversity` | Case 4 |
| UCB exploration for cold-start | Mode 1 — override `step_params.score` | Case 5 |
| Remove or change a pin | Mode 1 — override `step_params.pin.rules` | Case 6 |
| New step type in the pipeline | Mode 2 — full `vertical_pipeline` (BE registers step first) | Case 7 |

---

## What MLEs Own

- `p_act` block — predictor name, model name, features, calibration
- `step_params` overrides on any named step
- `output_config.emit_trace` — enable per-step tracing for this treatment

## What BE Owns (never touch these)

- Step execution order — the engine enforces it
- Sibyl RPC, retries, timeouts
- Feature fetching implementation (you declare names and sources, BE fetches)
- `pImp` position decay
- Trace schema and Kafka emission
- Step registry (you use registered step types; BE adds new ones)

---

## Case 1: New Primary Model Version

Swap the Sibyl predictor and model. All other pipeline params stay from control.

```json
{
  "treatment_fw_v4": {
    "extends": "control",
    "p_act": {
      "predictor_name": "FEED_RANKING_FW_SIBYL_PREDICTOR_NAME",
      "model_name": "store_ranker_fw_v4",
      "use_case": "USE_CASE_HOME_PAGE_VERTICAL_RANKING",
      "output_semantics": "PROBABILITY",
      "features": [
        { "name": "STORE_ID",         "source": "STORE" },
        { "name": "CONSUMER_ID",      "source": "CONSUMER" },
        { "name": "STORE_ETA",        "source": "STORE" },
        { "name": "DAY_OF_WEEK",      "source": "CONTEXT" },
        { "name": "IS_DASHPASS_USER", "source": "CONSUMER" }
      ],
      "calibration": { "type": "NONE" }
    }
  }
}
```

---

## Case 2: Adjust Vertical Boost Weight

Test multiple boost multipliers for NV (`vertical_id: 10`) in one file.
DV assigns users to a treatment name — two treatments = two DV buckets.

```json
{
  "treatment_nv_1_5x": {
    "extends": "control",
    "step_params": {
      "blend": {
        "vertical_boost_weights": {
          "boosting_multipliers": [{ "vertical_ids": [10], "multiplier": 1.5 }],
          "default_multiplier": 1.0
        }
      }
    }
  },
  "treatment_nv_2x": {
    "extends": "control",
    "step_params": {
      "blend": {
        "vertical_boost_weights": {
          "boosting_multipliers": [{ "vertical_ids": [10], "multiplier": 2.0 }],
          "default_multiplier": 1.0
        }
      }
    }
  }
}
```

Note: `step_params` shallow-merges. If you override `vertical_boost_weights`, you must
re-declare the full map — not just the key you're changing.

---

## Case 3: New Intent Model with Calibration

New model adds a consumer lifestyle cohort feature. Requires piecewise calibration because the
new model's score distribution differs from the control model's.

```json
{
  "treatment_intent_v3": {
    "extends": "control",
    "p_act": {
      "predictor_name": "FEED_RANKING_FW_SIBYL_PREDICTOR_NAME",
      "model_name": "vertical_intent_model_v3",
      "use_case": "USE_CASE_HOME_PAGE_VERTICAL_RANKING",
      "output_semantics": "PROBABILITY",
      "features": [
        { "name": "STORE_ID",            "source": "STORE" },
        { "name": "DAY_OF_WEEK",         "source": "CONTEXT" },
        { "name": "IS_DASHPASS_USER",    "source": "CONSUMER" },
        { "name": "NV_LIFESTAGE_COHORT", "source": "CONSUMER" }
      ],
      "calibration": {
        "type": "PIECEWISE",
        "params": {
          "calibration_entries": [
            {
              "vertical_ids": [2],
              "piecewise_multipliers": [
                { "min_score": 0.0, "max_score": 0.3, "multiplier": 1.2 },
                { "min_score": 0.3, "max_score": 1.0, "multiplier": 1.0 }
              ]
            }
          ],
          "default_multiplier": 1.0
        }
      }
    },
    "step_params": {
      "blend": {
        "intent_scoring_config": {
          "feature_configs": [
            { "feature_name": "day_of_week", "transform_fn": [{ "input": [1,7], "output": "weekend" }] },
            { "feature_name": "is_dashpass",  "transform_fn": [] }
          ],
          "score_lookup_entries": [
            {
              "feature_names": ["day_of_week", "is_dashpass"],
              "entries": { "weekend|1": 1.15, "weekday|1": 1.05 },
              "default_score": 1.0
            }
          ],
          "default_score": 1.0
        }
      }
    }
  }
}
```

---

## Case 4: Enable Diversity Reranking

Control has `diversity` step with `"enabled": false`. Override to turn it on.

```json
{
  "treatment_diversity_v1": {
    "extends": "control",
    "step_params": {
      "diversity": {
        "enabled": true,
        "weight": 0.4,
        "coefficient_for_non_rx": 0.6,
        "local_window_size_for_non_rx": 3,
        "local_coefficient_for_non_rx": 0.3
      }
    }
  }
}
```

Algorithm: greedy MMR-style rerank. For each slot, scores candidates by
`relevance_score + diversity_score` where diversity decays exponentially with consecutive
same-type content already placed. `weight` controls the relevance/diversity tradeoff.

---

## Case 5: UCB Exploration for Cold-Start

Add an exploration bonus to under-exposed NV carousels. Gradual ramp-up without hardcoded pins.

```json
{
  "treatment_nv_cold_start_ucb": {
    "extends": "control",
    "step_params": {
      "score": {
        "ucb_enabled": true,
        "ucb_predictor": "ucb_uncertainty",
        "ucb_model": "ucb_v2",
        "ucb_weight": 0.15
      }
    }
  }
}
```

The UCB bonus is proportional to model uncertainty for each carousel. High-uncertainty
(under-exposed) carousels get a lift. Fades naturally as exposure increases.

---

## Case 6: Measure and Remove a Legacy Pin

**Step 1** — Make the pin explicit in `control.json` (if not already). Enable tracing on the
treatment to capture `score_before` at the `FIXED_PINNING` step.

**Step 2** — Remove the pin in a treatment:

```json
{
  "treatment_no_pad_pin": {
    "extends": "control",
    "step_params": {
      "pin": { "rules": [] }
    },
    "output_config": { "emit_trace": true }
  }
}
```

Trace events show `score_before` at `FIXED_PINNING` = what PAD would have scored organically.
`score_after` = pinned position score. The gap is the opportunity cost.

Once you've measured and decided to remove, the treatment becomes the new control.

---

## Case 7: New Step Type in Pipeline (Mode 2)

Requires BE to implement and register the new step first. Then you declare the full pipeline.

```json
{
  "treatment_with_merch_boost": {
    "vertical_pipeline": {
      "steps": [
        { "id": "score",     "type": "MODEL_SCORING",    "params": { "predictor_ref": "p_act" } },
        { "id": "blend",     "type": "MULTIPLIER_BOOST", "params": { ... } },
        { "id": "diversity", "type": "DIVERSITY_RERANK", "params": { "enabled": true, "weight": 0.4 } },
        { "id": "merch",     "type": "MERCH_PLACEMENT",  "params": { "campaign_boost": 1.3 } },
        { "id": "pin",       "type": "FIXED_PINNING",    "params": { "rules": [] } }
      ]
    }
  }
}
```

`MERCH_PLACEMENT` must be registered in `buildStepRegistry()` by HP before this config can run.
After that one-time registration, any experiment can use it via config.

---

## Feature Sources

| `source` | Where BE fetches it |
|---|---|
| `STORE` | `DiscoveryStore` / `StoreEntity` fields |
| `CONSUMER` | `ConsumerContext` |
| `CONTEXT` | Request context (time, platform, device) |
| `GEO` | `GeoContext` |
| `FEATURE_STORE` | External feature store (requires `feature_store_key`) |

---

## Calibration Types

| `type` | When to use |
|---|---|
| `NONE` | Ranking within one content type, same model family — no calibration needed |
| `PIECEWISE` | Score-range → multiplier map. Use when new model has different score distribution. |
| `ISOTONIC` | Regression table for proper probability calibration. Use for cross-type ranking (post-POC). |

---

## Merge Semantics Reference

`extends: "control"` + `step_params`:
- Control pipeline is used as-is
- Each key in `step_params[stepId]` shallow-merges into that step's params
- Treatment wins on key collision; missing keys preserved from control
- Shallow merge: if you override a nested map (e.g. `vertical_boost_weights`), re-declare the full map

`vertical_pipeline` (Mode 2):
- Full pipeline takes complete precedence
- No merge with control
- Use only when step sequence itself changes
