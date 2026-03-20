# MLE Contract: Vertical Blending (Phase 1)

> The canonical MLE-facing document for configuring vertical (carousel-level) ranking experiments.
> Phase 1 scope only. Horizontal blending is Phase 2.

---

## What You Own vs. What BE Owns

```
MLE provides (in JSON)                BE platform provides (in code)
──────────────────────────────────    ──────────────────────────────────────
predictor_name + model_name           Sibyl RPC client + retry/timeout
feature list                          Feature fetching per source type
pipeline step sequence                Processor implementations + registry
step params (multipliers, weights)    Config loading + hot-reload
calibration config                    Calibration execution
emit_trace flag                       Trace schema + Kafka logging
```

The engine executes whatever you declare. No BE code changes for param changes.
BE changes code only when you need a new step type or new feature source.

---

## The Two Modes

### Mode 1: Override params only (90% of experiments)

Inherit the full control pipeline and change one or more step params.

```json
// ubp/experiments/{experiment_id}.json
{
  "treatment_fw_v4": {
    "extends": "control",
    "step_params": {
      "vertical_scoring": {
        "model_name": "store_ranker_fw_v4"
      }
    }
  }
}
```

Merge semantics: control pipeline is used as-is. Only the named step's params are overridden (shallow merge — you must re-declare all keys you want to keep if replacing a map value).

### Mode 2: Declare full pipeline (structural changes)

Use when the step sequence itself changes (adding/removing a step).

```json
{
  "treatment_merch_v1": {
    "value_function": { ... },
    "vertical_pipeline": {
      "steps": [
        { "id": "vertical_scoring",        "type": "MODEL_SCORING",  "params": { ... } },
        { "id": "vertical_boost_and_rank", "type": "BOOST_AND_RANK", "params": { ... } },
        { "id": "vertical_merch",          "type": "MERCH_PLACEMENT","params": { "campaign_boost": 1.3 } }
      ]
    }
  }
}
```

---

## The Control Baseline

HP owns `control.json`. All experiments that use `"extends": "control"` start from this.

```json
// ubp/experiments/control.json
{
  "control": {
    "value_function": {
      "p_act": {
        "predictor_name": "FEED_RANKING_SIBYL_PREDICTOR_NAME",
        "model_name": "store_ranking_v1_1",
        "use_case": "USE_CASE_HOME_PAGE_VERTICAL_RANKING",
        "output_semantics": "PROBABILITY",
        "features": [
          { "name": "STORE_ID",          "source": "STORE" },
          { "name": "CONSUMER_ID",       "source": "CONSUMER" },
          { "name": "DAY_OF_WEEK",       "source": "CONTEXT" },
          { "name": "IS_DASHPASS_USER",  "source": "CONSUMER" },
          { "name": "STORE_ETA",         "source": "STORE" }
        ],
        "calibration": { "type": "NONE" }
      }
    },
    "vertical_pipeline": {
      "steps": [
        {
          "id": "vertical_scoring",
          "type": "MODEL_SCORING",
          "params": { "predictor_ref": "p_act" }
        },
        {
          "id": "vertical_boost_and_rank",
          "type": "BOOST_AND_RANK",
          "params": {
            "boost_by_position_enabled": false,
            "deal_carousel_score_multiplier": 1.0,
            "boost_by_position_allow_list": []
          }
        }
      ]
    },
    "output_config": {
      "emit_trace": false,
      "max_vertical_components": 20
    }
  }
}
```

---

## Vertical Step Types (Phase 1)

### `MODEL_SCORING`

Calls Sibyl with the declared predictor, assigns raw score to each carousel component.

**Replaces:** `entityScorer.score()` in `EntityRankerConfiguration.getScoreBundleWithWorkflowHelper()`

```json
{
  "id": "vertical_scoring",
  "type": "MODEL_SCORING",
  "params": {
    "predictor_ref": "p_act"
  }
}
```

`predictor_ref` points to a key in `value_function` (always `"p_act"` for now). The engine resolves the full `PredictorConfig` (predictor name, model name, features, calibration) from there.

The `PredictorConfig` you declare in `value_function.p_act`:

```json
"p_act": {
  "predictor_name": "FEED_RANKING_FW_SIBYL_PREDICTOR_NAME",
  "model_name": "store_ranking_fw_v4",
  "use_case": "USE_CASE_HOME_PAGE_VERTICAL_RANKING",
  "output_semantics": "PROBABILITY",
  "features": [
    { "name": "STORE_ID",          "source": "STORE" },
    { "name": "DAY_OF_WEEK",       "source": "CONTEXT", "as_list_feature": true },
    { "name": "IS_DASHPASS_USER",  "source": "CONSUMER" }
  ],
  "calibration": { "type": "NONE" }
}
```

**Feature sources:**

| `source` | Where BE fetches it |
|---|---|
| `STORE` | `DiscoveryStore` / `StoreEntity` fields |
| `CONSUMER` | `ConsumerContext` |
| `CONTEXT` | Request context (time, platform, device) |
| `GEO` | `GeoContext` |
| `FEATURE_STORE` | External feature store (requires `feature_store_key`) |

---

### `BOOST_AND_RANK`

Wraps `getBoostBundle()` + `getRankingBundle()` + `getRankableContent()` — score assignment
to domain objects, position boosting, deal multiplier, pin vs flow sort order, reassembly.
All as one atomic flow.

**Replaces:** `Boosting.kt` + `BoostingBundle.boosted()` + `RankingBundle.ranked()`

```json
{
  "id": "vertical_boost_and_rank",
  "type": "BOOST_AND_RANK",
  "params": {
    "boost_by_position_enabled": true,
    "deal_carousel_score_multiplier": 1.5,
    "boost_by_position_allow_list": ["dt:some_carousel", "nv_carousel"]
  }
}
```

For experiments that only adjust boost params, use Mode 1:
```json
{
  "extends": "control",
  "step_params": {
    "vertical_boost_and_rank": {
      "boost_by_position_enabled": true,
      "deal_carousel_score_multiplier": 1.5,
      "boost_by_position_allow_list": ["dt:some_carousel"]
    }
  }
}
```

**Note:** Blending (calibration, intent, boost weights, diversity rerank) is now inside
`MODEL_SCORING`, not in a separate step. This is intentionally coarse — finer decomposition
is a future iteration.

---

## DV Wiring

One DV per experiment, following the naming convention:

```
DV label:     ubp_{experiment_id}
JSON path:    ubp/experiments/{experiment_id}.json
Map key:      the DV value (e.g. "control", "treatment_fw_v4")
```

Add one line to `DiscoveryExperimentManager.Manifest`:
```kotlin
UBP_HP_VERTICAL_FW_V4("ubp_hp_vertical_fw_v4"),
```

If the DV value doesn't match any key in the JSON, the engine silently falls back to `"control"`.

---

## Common Experiment Cases

### Case 1: New primary model

```json
{
  "treatment_fw_v4": {
    "extends": "control",
    "step_params": {
      "vertical_scoring": {}
    },
    "value_function": {
      "p_act": {
        "predictor_name": "FEED_RANKING_FW_SIBYL_PREDICTOR_NAME",
        "model_name": "store_ranking_fw_v4",
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
}
```

### Case 2: Adjust an NV boost weight

```json
{
  "treatment_nv_1_5x": {
    "extends": "control",
    "step_params": {
      "vertical_blend": {
        "vertical_boost_weights": {
          "boosting_multipliers": [{ "vertical_ids": [10], "multiplier": 1.5 }],
          "default_multiplier": 1.0
        }
      }
    }
  }
}
```

### Case 3: Enable diversity reranking

```json
{
  "treatment_diversity": {
    "extends": "control",
    "step_params": {
      "vertical_diversity": {
        "enabled": true,
        "diversity_weight": 0.4,
        "coefficient_for_non_rx": 0.6,
        "local_window_size_for_non_rx": 3,
        "local_coefficient_for_non_rx": 0.3
      }
    }
  }
}
```

### Case 4: New intent model (3 steps change together)

```json
{
  "treatment_intent_v3": {
    "extends": "control",
    "value_function": {
      "p_act": {
        "predictor_name": "FEED_RANKING_FW_SIBYL_PREDICTOR_NAME",
        "model_name": "vertical_intent_model_v3",
        "use_case": "USE_CASE_HOME_PAGE_VERTICAL_RANKING",
        "output_semantics": "PROBABILITY",
        "features": [
          { "name": "STORE_ID",           "source": "STORE" },
          { "name": "DAY_OF_WEEK",        "source": "CONTEXT" },
          { "name": "IS_DASHPASS_USER",   "source": "CONSUMER" },
          { "name": "NV_LIFESTAGE_COHORT","source": "CONSUMER" }
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
      }
    },
    "step_params": {
      "vertical_blend": {
        "intent_scoring_config": {
          "feature_configs": [
            { "feature_name": "day_of_week", "transform_fn": [{ "input": [1,7], "output": "weekend" }] },
            { "feature_name": "is_dashpass",  "transform_fn": [] }
          ],
          "score_lookup_entries": [
            {
              "feature_names": ["day_of_week", "is_dashpass"],
              "entries": { "weekend|1": 1.15 },
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

### Case 5: New pipeline step (requires BE to register a processor first)

```json
{
  "treatment_with_new_step": {
    "vertical_pipeline": {
      "steps": [
        { "id": "vertical_scoring",        "type": "MODEL_SCORING",  "params": { "predictor_ref": "p_act" } },
        { "id": "vertical_boost_and_rank", "type": "BOOST_AND_RANK", "params": { ... } },
        { "id": "merch",                   "type": "MERCH_PLACEMENT","params": { "campaign_boost": 1.3 } }
      ]
    }
  }
}
```

---

## Calibration Types

| Type | When to use | Params |
|---|---|---|
| `NONE` | Raw model output is good enough for ranking within this experiment | — |
| `PIECEWISE` | Per-vertical score range → multiplier (existing approach) | `calibration_entries`, `default_multiplier` |
| `ISOTONIC` | Proper probability calibration via isotonic regression | `runtime_path` (path to lookup table) |

Cross-vertical or cross-content-type comparisons require calibration. Within a single model experiment, `NONE` is fine.

---

## Output Config

```json
"output_config": {
  "emit_trace": true,
  "max_vertical_components": 20
}
```

`emit_trace: true` enables per-step score snapshots for all carousels. Cost: ~2x Iguazu event volume. Use in treatment arms; leave false in control to baseline traffic cost.

---

## Full Contract Reference

For the complete Kotlin data class schema and engine implementation:
`/Users/daniel.fonyo/Projects/feed-service/ubp-experiment-contract.md`
