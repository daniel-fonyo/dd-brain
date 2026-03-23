# MLE Contract — Vertical Blending (Phase 1)

> Canonical MLE-facing doc for configuring ranking experiments.
> Uses the simplest flat schema. Step order is hardcoded in the engine.

---

## Ownership

```
MLE provides (in JSON)                BE platform provides (in code)
──────────────────────────────────    ──────────────────────────────────────
p_act (predictor + model + features)  Sibyl RPC client + retry/timeout
v_act (value weights)                 Feature fetching per source type
boost_weights                         Engine step execution + registry
diversity params                      Config loading + hot-reload (5-min TTL)
output_config.emit_trace              Trace schema + Kafka logging
                                      pImp position decay (BE-only)
```

MLEs never touch: step order, Sibyl RPCs, feature fetching, pImp, trace schema.

---

## The Value Function

```
EV(c, k) = pImp(k) × pAct(c) × vAct(c)

pImp(k)  = P(user sees component at position k) — BE-owned, not in contract
pAct(c)  = P(user takes action | sees c) — Sibyl model output
vAct(c)  = Business value if action happens: gov_w × GOV + fiv_w × FIV + strategic_w × Strategic
```

Phase 1: `pAct` is explicit (MLE declares model). `vAct` and `pImp` are implicit (boost weights approximate value; position decay is future).

---

## Contract Schema

Step order is **hardcoded**: `MODEL_SCORING → BOOST_AND_RANK`. MLEs configure only the knobs.

| Field | Controls | Maps to formula |
|---|---|---|
| `p_act.predictor` + `p_act.model` | Which Sibyl model to call | pAct |
| `p_act.features` | What features to send to Sibyl | pAct inputs |
| `v_act` | Business outcome weights (GOV, FIV, strategic) | vAct |
| `boost_weights` | Per-vertical score multipliers | pAct post-processing |
| `diversity` | Whether diversity reranking runs and how strongly | pipeline param |

### Control Baseline

```json
{
  "control": {
    "p_act": {
      "predictor": "feed_ranking_v1",
      "model": "store_ranker_v1",
      "features": ["STORE_ID", "CONSUMER_ID", "STORE_ETA", "DAY_OF_WEEK", "IS_DASHPASS"]
    },
    "v_act": { "gov": 1.0, "fiv": 0.0, "strategic": 0.0 },
    "boost_weights": { "nv": 1.0 },
    "diversity": { "enabled": false, "weight": 0.0 }
  }
}
```

HP team owns control. MLEs do not touch it.

---

## Experiment Modes

### Mode 1: Override params (90% of experiments)

Inherit from control, change what differs:

```json
{
  "treatment_nv_boost": {
    "extends": "control",
    "v_act": { "gov": 0.8, "strategic": 0.2 },
    "boost_weights": { "nv": 1.5 }
  }
}
```

Merge semantics: shallow merge. Treatment value wins on collision. Re-declare full maps if overriding nested objects.

### Mode 2: Full pipeline (structural changes)

Use when adding a new step type (requires BE to register processor first):

```json
{
  "treatment_merch_v1": {
    "vertical_pipeline": {
      "steps": [
        { "id": "score", "type": "MODEL_SCORING", "params": { ... } },
        { "id": "boost_and_rank", "type": "BOOST_AND_RANK", "params": { ... } },
        { "id": "merch", "type": "MERCH_PLACEMENT", "params": { "campaign_boost": 1.3 } }
      ]
    }
  }
}
```

---

## Common Experiment Cases

### Case 1: New model version

```json
{
  "treatment_fw_v4": {
    "extends": "control",
    "p_act": {
      "predictor": "feed_ranking_fw_v1",
      "model": "store_ranker_fw_v4",
      "features": ["STORE_ID", "CONSUMER_ID", "STORE_ETA", "DAY_OF_WEEK", "IS_DASHPASS"]
    }
  }
}
```

### Case 2: NV strategic boost

```json
{
  "treatment_nv_boost_v2": {
    "extends": "control",
    "v_act": { "gov": 0.8, "fiv": 0.1, "strategic": 0.1 },
    "boost_weights": { "nv": 1.8 }
  }
}
```

### Case 3: New intent model + diversity

```json
{
  "treatment_intent_diversity": {
    "extends": "control",
    "p_act": {
      "predictor": "feed_ranking_fw_v1",
      "model": "vertical_intent_v3",
      "features": ["STORE_ID", "CONSUMER_ID", "DAY_OF_WEEK", "IS_DASHPASS", "NV_LIFESTAGE"]
    },
    "diversity": { "enabled": true, "weight": 0.4 }
  }
}
```

### Case 4: UCB exploration for cold-start

```json
{
  "treatment_nv_cold_start_ucb": {
    "extends": "control",
    "ucb": { "enabled": true, "predictor": "ucb_uncertainty", "model": "ucb_v2", "weight": 0.15 }
  }
}
```

### Case 5: Enable tracing to measure boost impact

```json
{
  "treatment_no_boost": {
    "extends": "control",
    "boost_weights": { "nv": 1.0 },
    "output_config": { "emit_trace": true }
  }
}
```

---

## DV Wiring

```
DV label:     ubp_{experiment_id}
JSON path:    ubp/experiments/{experiment_id}.json
Map key:      the DV value (e.g. "control", "treatment_fw_v4")
Fallback:     if DV value doesn't match any key → engine uses "control"
```

Add one line to `DiscoveryExperimentManager.Manifest`:
```kotlin
UBP_HP_VERTICAL_FW_V4("ubp_hp_vertical_fw_v4"),
```

---

## N × M Experiment Model

```
N experiments on horizontal ranking (Layer 3)
M experiments on vertical ranking (Layer 4)

DV "ubp_hp_horizontal_v2" → "treatment_fw"       (horizontal assignment)
DV "ubp_hp_vertical_v3"   → "intent_model__v3"   (vertical assignment)

Result: N × M valid cells. Independent per layer.
```

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

| Type | When to use |
|---|---|
| `NONE` | Ranking within one content type, same model family |
| `PIECEWISE` | Score-range → multiplier map (existing approach) |
| `ISOTONIC` | Proper probability calibration via isotonic regression (future) |

Cross-vertical or cross-content-type comparisons require calibration. Within a single model experiment, `NONE` is fine.

---

## What the Audit Trail Revealed

From real log dump (`consumer=757606047`):

| Expected | Found |
|---|---|
| Calibration multipliers vary | All = 1.0 (control has no calibration) |
| Intent multipliers vary | All = 1.0 (no intent model in control) |
| Boost weights are the lever | Confirmed: only active ranking signal |
| All 9 types flow through BlendingUtil | Only StoreCarousel + ItemCarousel — 7 types excluded via `else → null` |
| DealCarousel competes on same scale | False — excluded from blending, competes on raw Sibyl score |
| NV boost = 5.0× for carousel `07e18b95` | Got 1.25× — `business_vertical_id=169` lookup hit before `vertical_id=100322` |

Boost_weights lookup priority is `carousel_id` > `business_vertical_id` > `vertical_id` > default. Under UBP this is explicit in JSON — no hidden lookup priority bugs.

---

## Open Questions

- **VP**: dasher cost mode and commission fee models — how do these fit into the value function?
- **Exploration ranker**: runs after UR — should this be a separate step or part of MODEL_SCORING?
- **Score validation**: can we enforce that output scores make sense? Bounds checking?
- **Per-segment logic**: hardcoded DV per segment — how does runtime config handle this?
- **NV multi-score-head**: NV wants 7-day + same-day scores — how does the value function consume both?
- **GenAI carousel horizontal ranking**: when does VP value function get brought into GenAI?
- **Calibration workflow**: model ready → calibrate → param tune (2-3% traffic) → promote — is this a contract feature or an operational workflow?
- **Other teams providing model + predictor**: Promo, NV may also provide a different shard. How does this compose?

---

## Interview Questions (for MLE validation)

### Opening
1. What is the most painful part of running a new vertical ranking experiment today?
2. Which config params do you change most often?
3. Have you ever wanted to run an experiment but couldn't because it required a code change?

### What MLEs actually change
4. When you launch a new model, what exactly do you want to swap?
5. How do you currently set boost weights for NV vs Rx carousels?
6. Do you care about diversity reranking as an MLE knob?
7. For value function weights (GOV, FIV, strategic) — per experiment or business policy?
8. What does "calibration" mean to you?

### Experiment structure
9. Full config per experiment, or inherit from control and override?
10. Do you run N × M experiments across ranking layers?
11. Would two treatments ever share a base that isn't control?

### Observability
12. What traces do you look at after launching an experiment?
13. Do you care about per-step score snapshots?
14. How do you debug ranking issues today?

### Contract validation
15. Does this contract capture everything for your next experiment?
16. Params you never want to configure (pImp, RPC retries)?
17. Rollback story — does "change DV back to control" cover it?
