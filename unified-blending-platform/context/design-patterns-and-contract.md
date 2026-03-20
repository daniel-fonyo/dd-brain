# UBP: Design Patterns & MLE Contract

---

## Design Patterns

### Strategy — core pattern for processors

Each `VerticalProcessor` is a Strategy. The engine doesn't know or care which algorithm runs — it just calls `process()`. Swapping a boost strategy or scoring strategy means registering a different implementation, not changing the engine.

```
VerticalProcessor (interface)
    ├── ModelScoringProcessor
    └── BoostAndRankProcessor
```

The config JSON picks which strategies run and in what order. The engine dispatches.

---

### Chain of Responsibility — pipeline execution

Each step receives the component list, does its work, and passes it to the next. The step order is **hardcoded in the engine** — it is determined by the semantics of each step and never needs to change:

```
components → [MODEL_SCORING] → [BOOST_AND_RANK] → sorted result
```

There is no valid experiment where this order would be different: you can't boost before you have scores. MLEs configure params within each step, not the sequence itself.

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

## The Value Function

This is where all the variables come together. The engine computes this for every component (carousel or store) at every position:

```
EV(c, k) = pImp(k) × pAct(c) × vAct(c)
```

**Variable dictionary:**

| Abbrev | Full name | What it is | Who owns it |
|---|---|---|---|
| `EV` | Expected Value | Final ranking score — higher = shown earlier | Engine output |
| `pImp(k)` | Probability of Impression | P(user sees component at position k). Decays with position: `decay_rate ^ k` | BE — not MLE-configurable |
| `pAct(c)` | Probability of Action | P(user clicks/orders given they see c). The Sibyl model output. | MLE provides model + features |
| `vAct(c)` | Value of Action | Business worth if action happens: `gov_w × GOV + fiv_w × FIV + strategic_w × Strategic` | MLE provides weights |
| `GOV` | Gross Order Value | Revenue from the order | BE signal |
| `FIV` | First Impression Value | Strategic value of first-time NV/vertical exposure | BE signal |
| `k` | Position index | 0-indexed row (vertical) or slot (horizontal) | — |
| `c` | Component | A carousel (vertical) or store/item (horizontal) | — |

**Where each variable is computed in the pipeline:**

```
MODEL_SCORING     → calls Sibyl → BlendingUtil.blendBundle() including diversity rerank
                    → writes pAct(c) × vAct(c) approximation to component.score
BOOST_AND_RANK    → score assignment to domain objects, position boosting, deal multiplier,
                    pin vs flow sort order, reassembly
final sort        → sorts by component.score descending = sorts by EV descending
```

Today `vAct` and `pImp` are implicit — boost weights stand in for value weights with no principled formula. Phase 1 makes `pAct` explicit (MLE declares the model). Phase 3 makes `vAct` and `pImp` explicit (configurable weights + position decay model).

---

## Vertical MLE Contract — Minimal JSON

The formula is: **Expected Value = pImp × pAct × vAct**

Step order is hardcoded in the engine. MLEs configure only the knobs:

| Field | Controls | Maps to formula |
|---|---|---|
| `p_act.model` + `p_act.features` | Which Sibyl model to call and what to send it | pAct |
| `v_act` | Business outcome weights (GOV, FIV, strategic) | vAct |
| `boost_weights` | Per-vertical score multipliers | pAct post-processing |
| `diversity` | Whether diversity reranking runs and how strongly | pipeline param |

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
  },

  "treatment_nv_boost": {
    "extends": "control",
    "v_act": { "gov": 0.8, "strategic": 0.2 },
    "boost_weights": { "nv": 1.5 }
  },

  "treatment_intent_v3": {
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

The engine always runs: `MODEL_SCORING → BOOST_AND_RANK`.
MLEs never configure the sequence — they configure what each step does.

### What MLEs never touch

- Step order — hardcoded by the engine
- Sibyl RPC, retries, timeouts — BE owns
- Feature fetching logic — BE owns
- pImp (position decay) — BE owns
- Trace schema + logging — BE owns, auto-emitted after every step

---

## Horizontal Contract (Phase 2)

Same formula, same pattern — applied to store/item ordering *within* each carousel.

The key difference: carousels have different ranking needs. Favorites needs order-history reranking; NV needs a different boost; Taste just needs model score. The contract supports this via `carousel_overrides` — a per-carousel step override on top of the default pipeline.

### Current code problems (mirror of vertical)

| Root cause | Current code | UBP fix |
|---|---|---|
| Each carousel type owns its ranking logic | `FavoritesCarousel`, `DVCarousel`, `TasteCarousel` each have bespoke `when` chains in `modifyLiteStoreCollection()` | `HorizontalProcessor` registry + `carousel_overrides` in config |
| S1/S2/S3 reranking is invisible | `rerankDecoratedEntities()` runs after Iguazu fires — logged positions ≠ positions users see | `RANKING_SORT` step (includes order history rerank), auto-traced like any other |
| Same config fragmentation | Predictor selection across 5+ DVs, feature construction hardcoded in `StoreCollectionScorer` | Same `p_act` + `steps` contract as vertical |

### Horizontal step types

| Step type | What it wraps | Replaces in current code |
|---|---|---|
| `MODEL_SCORING` | `scoredCollections()` — Sibyl + score modifiers (atomic) | `StoreCollectionScorer` |
| `RANKING_SORT` | `modifyLiteStoreCollection()` `when(rankingType)` dispatch | `DefaultHomePageStoreRanker.modifyLiteStoreCollection()` when-chain |

### Minimal horizontal JSON

Step order is hardcoded: `MODEL_SCORING → RANKING_SORT`.
`carousel_overrides` lets specific carousels configure different ranking-type-specific params.

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
    "order_history_rerank": { "enabled": false },
    "carousel_overrides": {
      "FAVORITES": {
        "order_history_rerank": { "enabled": true, "tiers": ["S1", "S2", "S3"], "lookback": 10 }
      }
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
    "boost_weights": { "nv": 1.8 }
  }
}
```

### `carousel_overrides` semantics

Per-carousel param overrides on top of the defaults. The engine applies the default config first, then merges the carousel-specific override for that carousel type. This keeps carousel-specific logic (S1/S2/S3 only on FAVORITES, NV boost only on NV) declared explicitly rather than hardcoded inside carousel classes.

Order history reranking (now part of the `RANKING_SORT` step) is notable: today it runs silently after Iguazu fires, so logged positions ≠ positions users see. Under UBP it's inside a declared step — auto-traced, logged positions are accurate.
