# UBP: Design Patterns & MLE Contract

---

## Design Patterns

### Strategy — core pattern for processors

Each `VerticalProcessor` is a Strategy. The engine doesn't know or care which algorithm runs — it just calls `process()`. Swapping a boost strategy or scoring strategy means registering a different implementation, not changing the engine.

```
VerticalProcessor (interface)
    ├── ModelScoringProcessor
    ├── MultiplierBoostProcessor
    ├── DiversityRerankProcessor
    └── FixedPinningProcessor
```

The config JSON picks which strategies run and in what order. The engine dispatches.

---

### Chain of Responsibility — pipeline execution

Each step in the pipeline receives the component list, does its work, and passes it to the next step. Each processor is a handler in the chain. The chain is defined by the config, not hardcoded.

```
components → [MODEL_SCORING] → [MULTIPLIER_BOOST] → [DIVERSITY_RERANK] → [FIXED_PINNING] → sorted result
```

Unlike the classic pattern, every handler in the chain always runs (no early termination). The chain is assembled from config at request time.

---

### Adapter — wrapping carousel types behind one interface

The 9 existing carousel types (`StoreCarousel`, `ItemCarousel`, `DealCarousel`, etc.) are adapted behind a single `VerticalComponent` interface. Processors only ever see `VerticalComponent` — no type-switching inside the engine.

```
StoreCarousel  ──┐
ItemCarousel   ──┤ Adapter → VerticalComponent (id, type, score, metadata)
DealCarousel   ──┤
...            ──┘
```

After ranking, scores are written back to the original domain objects via `applyBackTo()`.

---

### Template Method — what the current code uses (the pattern being replaced)

`BaseEntityRankerConfiguration.rank()` defines a fixed skeleton: score → blend → boost → sort. Subclasses override steps but cannot reorder or skip them.

UBP replaces this with a config-driven Chain of Responsibility. The template method becomes the engine's `rank()` loop — but the step sequence comes from JSON, not inheritance.

---

### Registry — processor lookup

`processorRegistry: Map<String, VerticalProcessor>` maps step type strings to implementations. Adding a new step type = adding one entry to the registry. Nothing else changes.

This is the mechanism that makes the Chain of Responsibility config-driven: the config names the steps, the registry resolves them.

---

### Facade — engine as single entry point

`VerticalRankingEngine.rank()` hides the complexity of config resolution, processor dispatch, and auto-tracing behind one method call. Callers (`DefaultHomePagePostProcessor`) don't know about the registry, the config schema, or individual processors.

---

## MLE Contract — Minimal JSON

The formula is: **Expected Value = pImp × pAct × vAct**

- `pImp` — position decay, owned by BE (not MLE-configurable)
- `pAct` — action probability, **MLE provides the model**
- `vAct` — value of action, **MLE provides the weights**
- pipeline steps — **MLE declares which processors run and with what params**

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
    "steps": [
      { "type": "MODEL_SCORING" },
      { "type": "MULTIPLIER_BOOST", "boost_weights": { "nv": 1.0 } },
      { "type": "DIVERSITY_RERANK", "enabled": false },
      { "type": "FIXED_PINNING",    "pins": [] }
    ]
  },

  "treatment_nv_boost": {
    "extends": "control",
    "v_act": { "gov": 0.8, "strategic": 0.2 },
    "step_params": {
      "MULTIPLIER_BOOST": { "boost_weights": { "nv": 1.5 } }
    }
  },

  "treatment_intent_v3": {
    "extends": "control",
    "p_act": {
      "predictor": "feed_ranking_fw_v1",
      "model": "vertical_intent_v3",
      "features": ["STORE_ID", "CONSUMER_ID", "DAY_OF_WEEK", "IS_DASHPASS", "NV_LIFESTAGE"]
    },
    "step_params": {
      "DIVERSITY_RERANK": { "enabled": true, "weight": 0.4 }
    }
  }
}
```

### What each field does

| Field | Controls | Maps to formula |
|---|---|---|
| `p_act.predictor` + `p_act.model` | Which Sibyl endpoint and model version to call | pAct |
| `p_act.features` | What features to send to Sibyl | pAct inputs |
| `v_act.gov/fiv/strategic` | How to weight business outcomes in the score | vAct |
| `steps[].type` | Which processors run and in what order | pipeline shape |
| `step_params` | Tune a specific processor without changing the pipeline | processor params |

### What MLEs never touch

- Sibyl RPC, retries, timeouts — BE owns
- Feature fetching logic — BE owns
- pImp (position decay) — BE owns, not an experiment variable
- Trace/logging schema — BE owns, auto-emitted by engine
- Processor implementations — BE owns

### The `extends` shortcut

`"extends": "control"` copies the full control pipeline then applies only the declared overrides. This is the common case — no need to re-declare every step for a small change.

If the step sequence itself changes (adding/removing a step), declare the full `steps` array instead.

---

## Horizontal Contract (Phase 2)

Same formula, same pattern — applied to store/item ordering *within* each carousel.

The key difference: carousels have different ranking needs. Favorites needs order-history reranking; NV needs a different boost; Taste just needs model score. The contract supports this via `carousel_overrides` — a per-carousel step override on top of the default pipeline.

### Current code problems (mirror of vertical)

| Root cause | Current code | UBP fix |
|---|---|---|
| Each carousel type owns its ranking logic | `FavoritesCarousel`, `DVCarousel`, `TasteCarousel` each have bespoke `when` chains in `modifyLiteStoreCollection()` | `HorizontalProcessor` registry + `carousel_overrides` in config |
| S1/S2/S3 reranking is invisible | `rerankDecoratedEntities()` runs after Iguazu fires — logged positions ≠ positions users see | `ORDER_HISTORY_RERANK` step, auto-traced like any other |
| Same config fragmentation | Predictor selection across 5+ DVs, feature construction hardcoded in `StoreCollectionScorer` | Same `p_act` + `steps` contract as vertical |

### Horizontal step types

| Step type | What it does | Replaces in current code |
|---|---|---|
| `MODEL_SCORING` | Call Sibyl, assign score to each store/item | `StoreCollectionScorer.regressionContext()` |
| `MULTIPLIER_BOOST` | Strategic boost (NV, DashPass, campaign) | `getScoreModifierMap()` inside scorer |
| `BUSINESS_RULES_SORT` | Apply open/closed, priority biz, campaign pin rules | `modifyLiteStoreCollection()` when-chain |
| `ORDER_HISTORY_RERANK` | S1/S2/S3 tier reranking by order history | `FavoritesCarousel.rerankDecoratedEntities()` |

### Minimal horizontal JSON

```json
{
  "control": {
    "p_act": {
      "predictor": "feed_ranking_v1",
      "model": "store_ranker_v1",
      "features": ["STORE_ID", "CONSUMER_ID", "STORE_ETA", "DAY_OF_WEEK", "IS_DASHPASS"]
    },
    "v_act": { "gov": 1.0, "fiv": 0.0, "strategic": 0.0 },
    "default_steps": [
      { "type": "MODEL_SCORING" },
      { "type": "MULTIPLIER_BOOST", "boost_weights": { "nv": 1.0 } },
      { "type": "BUSINESS_RULES_SORT" }
    ],
    "carousel_overrides": {
      "FAVORITES": [
        { "type": "MODEL_SCORING" },
        { "type": "BUSINESS_RULES_SORT" },
        { "type": "ORDER_HISTORY_RERANK", "tiers": ["S1", "S2", "S3"], "lookback": 10 }
      ]
    }
  },

  "treatment_fw_v2": {
    "extends": "control",
    "p_act": {
      "predictor": "feed_ranking_fw_v1",
      "model": "store_ranker_fw_v2",
      "features": ["STORE_ID", "CONSUMER_ID", "STORE_ETA", "SCENE_ID", "CONTAINER_CODE", "IS_DASHPASS"]
    }
  },

  "treatment_nv_strategic": {
    "extends": "control",
    "v_act": { "gov": 0.7, "strategic": 0.3 },
    "step_params": {
      "MULTIPLIER_BOOST": { "boost_weights": { "nv": 1.8 } }
    }
  }
}
```

### `carousel_overrides` semantics

If a carousel type has an entry in `carousel_overrides`, that step list replaces `default_steps` entirely for that carousel. This is how carousel-specific logic (S1/S2/S3, NV-only boost) is declared without polluting the default pipeline.

The S1/S2/S3 `ORDER_HISTORY_RERANK` step is notable: today it runs silently after logging fires, so the logged positions are wrong. Under UBP it's a declared step — engine traces it, logged positions match what users see.
