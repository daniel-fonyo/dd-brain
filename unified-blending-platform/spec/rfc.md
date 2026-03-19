# UBP Platform Contracts

> The five contracts that define the Unified Blending Platform boundary. Each contract has a
> counterparty — the team that writes or implements it. Together they eliminate the coupling
> between HP engineering and partner teams.

---

## Why Five Contracts

Every pain point in the UBP vision traces to one of two root causes:

1. **No clean API boundary** — HP must understand domain logic (NV, Ads, Merch) to incorporate it
2. **No common scoring scale** — organic, ads, and merch scores are incomparable, so ranking cannot be global

These five contracts establish the boundary:

| # | Contract | Counterparty | Root cause addressed |
|---|---|---|---|
| 1 | Experiment Config (JSON) | MLEs, partner teams | No API boundary — config hell |
| 2 | FeedRow / RowItem Interface | Any team whose content gets ranked | No common unit in the pipeline |
| 3 | Value Function / Score | Model teams (UR, Ads, VP ranker) | No common scoring scale |
| 4 | RankingStep Interface | Domain teams (NV, Ads, Merch) | No extension point without HP involvement |
| 5 | Trace Event | Analytics / data science | No stage-wise observability |

---

## Contract 2: FeedRow / RowItem Interface

**Counterparty: Any team whose content wants to be ranked**

Today there are 9+ separate typed carousel classes with no shared interface. Every operation on
"all carousels" branches on type. Processors, scorers, and dedup logic all carry the same
type-switching glue.

```kotlin
interface FeedRow {
    val id: String
    val type: RowType          // STORE_CAROUSEL, ITEM_CAROUSEL, NV_CAROUSEL, ... (see RowType enum)
    var score: Double                // mutable — processors update this in-place
    val metadata: MutableMap<String, Any>
    fun recordTrace(stepId: String, value: Any)
    fun applyBackTo()                // writes final score back to the underlying domain object
}
```

The engine operates on `MutableList<FeedRow>` throughout. This list contains organic carousels
from `HomePageStoreLayoutOutputElements`. Ads are not FeedRows — they blend at the store ranker
level as `RowItem` objects within each carousel (Phase 1.5 horizontal ranking). There is no
`AD_CAROUSEL` FeedRow.

To participate in UBP ranking, a content type implements this interface and provides an adapter:

```kotlin
class StoreCarouselRow(val carousel: StoreCarousel) : FeedRow {
    override val id = carousel.id
    override val type = RowType.STORE_CAROUSEL
    override var score: Double = carousel.predictionScore ?: 0.0
    override fun applyBackTo() { carousel.sortOrder = score.toInt() }
}
```

**Pain points addressed:** 1.1 (fragmented optimization), 2.3 (post-organic ads insertion),
3.3 (Merch reinventing wheels — they implement adapters, not a parallel engine)

---

## Contract 3: Value Function / Score Contract

**Counterparty: Model teams (Universal Ranker, Ads ranking, VP ranker)**

The formula every content type's score is expressed in:

```
Expected Value = pImp(k) × pAct(c) × vAct(c)

  pImp(k)  = P(user sees component at position k)
             Decays with position: decay_rate^k
             BE owns — not MLE-configurable

  pAct(c)  = P(user takes action | they see c)
             The ML model output (CTR, CVR, order probability)
             MLE provides model + features + calibration spec

  vAct(c)  = Value of that action in business terms
             gov_w × GOV + fiv_w × FIV + strategic_w × Strategic
             MLE provides weights per experiment
```

For two content types to compete for the same position, their scores must be on a comparable
calibrated scale. Today this is impossible: UR scores are engagement-based, ads scores are
eCPM/pCTR, NV scores are heuristic multipliers.

Calibration types a model team declares in their experiment config:

| Type | When to use |
|---|---|
| `NONE` | Ranking within one content type, same model family |
| `PIECEWISE` | Per-vertical score range → multiplier (existing approach) |
| `ISOTONIC` | Proper probability calibration via isotonic regression lookup table |

**Phase roadmap:**
- Phase 1: `pAct` explicit (MLE declares the model). `vAct` approximated by boost weights.
- Phase 3: `vAct` explicit (configurable `gov_w`, `fiv_w`, `strategic_w`). `pImp` explicit (position decay config).

**Pain points addressed:** 1.3 (incomparable scoring scales), 2.1 (no systematic value function),
2.5 (opportunity cost blindness — pinned components have a measurable displaced value)

---

## Contract 4: RankingStep Interface

**Counterparty: Domain teams that need custom ranking logic**

HP owns the engine and registry. Partner teams own their processors.

```kotlin
interface FeedRowRankingStep {
    val type: String   // registered key referenced in experiment JSON steps

    suspend fun process(
        components: MutableList<FeedRow>,
        context: RankingContext,
        params: Map<String, Any>,   // injected from config at call time — no internal DV reads
    )
}
```

`params` injected at call time is the critical property. A processor that reads its own DV keys
internally is not a UBP processor — it defeats the entire point. All tunable behavior must flow
through `params`.

Adding a new step type:
1. Team implements `FeedRowRankingStep`
2. HP registers it in the step registry (one line)
3. Any experiment can use it via config immediately

**Example:** NV team wants custom trial-ramp logic. They implement `NvTrialBoostStep`,
HP registers it. From then on NV configures it via JSON — HP never understands NV business logic.

```kotlin
class NvTrialBoostStep : FeedRowRankingStep {
    override val type = "NV_TRIAL_BOOST"

    override suspend fun process(
        components: MutableList<FeedRow>,
        context: RankingContext,
        params: Map<String, Any>,
    ) {
        val boostMultiplier = params["boost_multiplier"] as Double
        val maxPosition = params["max_position"] as Int
        components
            .filter { it.type == RowType.NV_CAROUSEL }
            .take(maxPosition)
            .forEach { it.score *= boostMultiplier }
    }
}
```

**Pain points addressed:** 3.2 (ownership ambiguity — clear boundary: HP owns engine, partners own
processors), 3.3 (Merch can own `MerchPlacementStep`), 3.4 (domain expertise bottleneck gone)

---

## Contract 5: Trace Event

**Counterparty: Analytics and data science**

Standard Iguazu event emitted automatically after every pipeline step. No manual instrumentation.

```protobuf
message UbpVerticalRankingTrace {
  string request_id     = 1;
  string experiment_id  = 2;
  string treatment      = 3;
  string step_id        = 4;
  string row_id   = 5;
  string row_type = 6;
  double score_before   = 7;
  double score_after    = 8;
  int64  timestamp_ms   = 9;
}
```

Enabled per experiment via `output_config.emit_trace: true`. Disabled in control to baseline
traffic cost.

**What this enables:**
- Stage-wise score snapshots queryable in Snowflake without code changes
- Counterfactuals: what would have ranked differently if boost weight were 1.0?
- Opportunity cost of pinning: `score_before` on a `FIXED_PINNING` step = organic value displaced
- Validating that model changes are actually affecting scores (not silently ignored)
- Retiring legacy heuristics: measure impact before removing

**Pain points addressed:** 1.4 (observability gaps), 2.2 (legacy heuristics without measurement),
2.5 (opportunity cost blindness)

---

## Contract 1: Experiment Config (JSON)

**Counterparty: MLEs and partner teams (NV, Ads, Merch)**

The primary self-service interface. Replaces DV configuration hell. An MLE drops a JSON file and
adds one DV manifest entry — no HP engineering required, no 15+ file changes, no 2-week setup.

---

### File Structure

```
ubp/experiments/{experiment_id}.json   ← one file per experiment
ubp/experiments/control.json           ← HP owns this; the baseline all treatments inherit from
```

One DV per experiment:
```
DV label:   ubp_{experiment_id}
JSON key:   the DV value (e.g. "treatment_fw_v4", "treatment_nv_1_5x")
Fallback:   if DV value doesn't match any key → engine uses "control" silently
```

One line in `DiscoveryExperimentManager.Manifest`:
```kotlin
UBP_HP_VERTICAL_FW_V4("ubp_hp_vertical_fw_v4"),
```

Hot-reloaded via `DeserializedRuntimeCache` (5-min TTL). No pod restart for param changes.

---

### The Two Modes

#### Mode 1 — Param override (90% of experiments)

Inherit the full control pipeline, change one or more step params.

```json
{
  "treatment_fw_v4": {
    "extends": "control",
    "step_params": {
      "score": { "primary_model": "store_ranker_fw_v4" }
    }
  }
}
```

Merge semantics: control pipeline is used as-is. Each key in `step_params` shallow-merges into
that step's params (treatment wins on collision). Steps not mentioned are unchanged.
Re-declare all keys you want to keep if replacing a map value.

#### Mode 2 — Full pipeline declaration (structural changes)

Use when the step sequence itself changes (adding/removing a step).

```json
{
  "treatment_diversity_v1": {
    "vertical_pipeline": {
      "steps": [
        { "id": "score",     "type": "MODEL_SCORING",    "params": { ... } },
        { "id": "blend",     "type": "MULTIPLIER_BOOST", "params": { ... } },
        { "id": "diversity", "type": "DIVERSITY_RERANK", "params": { "weight": 0.4 } },
        { "id": "pin",       "type": "FIXED_PINNING",    "params": { ... } }
      ]
    }
  }
}
```

Full declaration takes complete precedence — no merge with control.

---

### The N × M Experiment Model

Horizontal and vertical experiments are independent. A user is assigned to one horizontal variant
AND one vertical variant simultaneously, giving N × M valid experiment cells.

```
DV "ubp_hp_horizontal_v2" → "treatment_fw"         (horizontal assignment)
DV "ubp_hp_vertical_v3"   → "intent_model_v3"       (vertical assignment)
```

Each engine reads its own DV independently. No interference. Experiments are mutually exclusive
*within* a layer, not *across* layers.

---

### Control Baseline

HP owns `control.json`. It canonically defines production behavior. All `extends: "control"`
experiments start here.

```json
{
  "control": {
    "p_act": {
      "predictor_name": "FEED_RANKING_SIBYL_PREDICTOR_NAME",
      "model_name": "store_ranking_v1_1",
      "use_case": "USE_CASE_HOME_PAGE_VERTICAL_RANKING",
      "output_semantics": "PROBABILITY",
      "features": [
        { "name": "STORE_ID",         "source": "STORE" },
        { "name": "CONSUMER_ID",      "source": "CONSUMER" },
        { "name": "DAY_OF_WEEK",      "source": "CONTEXT" },
        { "name": "IS_DASHPASS_USER", "source": "CONSUMER" },
        { "name": "STORE_ETA",        "source": "STORE" }
      ],
      "calibration": { "type": "NONE" }
    },
    "v_act": { "gov": 1.0, "fiv": 0.0, "strategic": 0.0 },
    "vertical_pipeline": {
      "steps": [
        {
          "id": "score",
          "type": "MODEL_SCORING",
          "params": { "predictor_ref": "p_act" }
        },
        {
          "id": "blend",
          "type": "MULTIPLIER_BOOST",
          "params": {
            "calibration_config": {
              "calibration_entries": [],
              "default_multiplier": 1.0
            },
            "intent_scoring_config": {
              "feature_configs": [],
              "score_lookup_entries": [],
              "default_score": 1.0
            },
            "vertical_boost_weights": {
              "boosting_multipliers": [],
              "default_multiplier": 1.0
            }
          }
        },
        {
          "id": "diversity",
          "type": "DIVERSITY_RERANK",
          "params": { "enabled": false, "weight": 0.0 }
        },
        {
          "id": "pin",
          "type": "FIXED_PINNING",
          "params": {
            "rules": [
              { "row_id": "pad",            "position": 3, "hard_pin": true },
              { "row_id": "member_pricing", "position": 0, "hard_pin": true }
            ]
          }
        }
      ]
    },
    "output_config": {
      "emit_trace": false,
      "max_feed_rows": 20
    }
  }
}
```

---

### Vertical Step Types

#### `MODEL_SCORING`

Calls Sibyl with the declared predictor, assigns raw `pAct` score to each component.

```json
{
  "id": "score",
  "type": "MODEL_SCORING",
  "params": {
    "predictor_ref": "p_act"
  }
}
```

`predictor_ref` resolves to a key in the top-level `p_act` block. The engine reads
`predictor_name`, `model_name`, `use_case`, `features`, and `calibration` from there.

**Feature sources:**

| `source` | Where BE fetches it |
|---|---|
| `STORE` | `DiscoveryStore` / `StoreEntity` fields |
| `CONSUMER` | `ConsumerContext` |
| `CONTEXT` | Request context (time, platform, device) |
| `GEO` | `GeoContext` |
| `FEATURE_STORE` | External feature store (requires `feature_store_key`) |

**Calibration types:**

| `type` | When to use |
|---|---|
| `NONE` | Raw model output sufficient; ranking within single content type |
| `PIECEWISE` | Per-vertical score range → multiplier map |
| `ISOTONIC` | Proper probability calibration via isotonic regression lookup table |

Cross-vertical or cross-content-type score comparisons require `PIECEWISE` or `ISOTONIC`.

**UCB/exploration variant:**

```json
{
  "id": "score",
  "type": "MODEL_SCORING",
  "params": {
    "predictor_ref": "p_act",
    "ucb_enabled": true,
    "ucb_predictor": "ucb_uncertainty",
    "ucb_model": "ucb_v2",
    "ucb_weight": 0.1
  }
}
```

UCB adds an exploration bonus proportional to model uncertainty. Useful for cold-starting new
verticals or new content types without pinning.

**MAB variant:**

```json
{
  "id": "score",
  "type": "MODEL_SCORING",
  "params": {
    "predictor_ref": "p_act",
    "mab_enabled": true,
    "mab_model": "mab_slot_v1"
  }
}
```

MAB enables multi-armed bandit selection among scored carousels.

---

#### `MULTIPLIER_BOOST`

Applies calibration × intent multiplier × vertical boost weights to each component's score.
Wraps `BlendingUtil.blendBundle()`.

```json
{
  "id": "blend",
  "type": "MULTIPLIER_BOOST",
  "params": {
    "calibration_config": {
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
    },
    "intent_scoring_config": {
      "feature_configs": [
        {
          "feature_name": "day_of_week",
          "transform_fn": [{ "input": [1, 7], "output": "weekend" }]
        },
        {
          "feature_name": "is_dashpass",
          "transform_fn": []
        }
      ],
      "score_lookup_entries": [
        {
          "feature_names": ["day_of_week", "is_dashpass"],
          "entries": { "weekend|1": 1.15, "weekday|1": 1.05 },
          "default_score": 1.0
        }
      ],
      "default_score": 1.0
    },
    "vertical_boost_weights": {
      "boosting_multipliers": [
        { "vertical_ids": [2],  "multiplier": 1.0 },
        { "vertical_ids": [10], "multiplier": 1.2 }
      ],
      "default_multiplier": 1.0
    }
  }
}
```

`vertical_ids` reference business vertical IDs (e.g. 2 = Rx, 10 = NV).

**Value function integration (Phase 3):** When `v_act` weights are declared, `MULTIPLIER_BOOST`
multiplies each component score by `gov_w × GOV(c) + fiv_w × FIV(c) + strategic_w × Strategic(c)`.
In Phase 1 the boost weights stand in for `vAct` without the principled formula.

---

#### `DIVERSITY_RERANK`

Greedy rerank that penalizes non-Rx density using exponential decay with a local sliding window.
Wraps `BlendingUtil.rerankEntitiesWithDiversity()`.

```json
{
  "id": "diversity",
  "type": "DIVERSITY_RERANK",
  "params": {
    "enabled": true,
    "weight": 0.4,
    "coefficient_for_non_rx": 0.6,
    "local_window_size_for_non_rx": 3,
    "local_coefficient_for_non_rx": 0.3
  }
}
```

Set `"enabled": false` to disable without removing the step. The step is still traced either way.

**Algorithm:** For each position, computes:
- `topRxScore = decayedDiversityWeight`
- `topNonRxScore = decayedDiversityWeight × exp(−coefficient × totalNonRx − localCoefficient × localNonRx)`

Selects entity with highest `relevanceScore + diversityScore`. Prevents pages dominated by one vertical.

---

#### `FIXED_PINNING`

Pins a component to a specific page position by inflating its score above everything at that
position. Wraps `BoostingBundle.boosted()` + all `NonRankableHomepageOrderingUtil` post-ranking
fixups. This makes today's silent post-ranking overrides declared, visible, and traced.

```json
{
  "id": "pin",
  "type": "FIXED_PINNING",
  "params": {
    "rules": [
      {
        "row_id": "pad",
        "position": 3,
        "hard_pin": true,
        "condition": "component_present"
      },
      {
        "row_id": "member_pricing",
        "position": 0,
        "hard_pin": true,
        "condition": "component_present"
      },
      {
        "row_id": "nv_trial",
        "position": 2,
        "hard_pin": false,
        "condition": "is_post_checkout_nv_boosting"
      }
    ]
  }
}
```

Multiple rules in one step. Order of declaration determines precedence on conflict.
`hard_pin: true` = unconditional. `hard_pin: false` = conditional on `condition` evaluating true.

Today's fixups that become declared `FIXED_PINNING` rules:
- `updateSortOrderOfNVCarousel()` → `row_id: "nv_trial", condition: "is_post_checkout_nv_boosting"`
- `updateSortOrderOfTasteOfDashPass()` → `row_id: "pad", position: 3`
- `updateSortOrderOfMemberPricing()` → `row_id: "member_pricing", position: 0`

---

#### `CALIBRATION` (standalone step, Phase 3)

Transforms raw model scores onto a common calibrated scale. Enables organic vs ads vs merch
to compete for the same positions.

```json
{
  "id": "calibrate",
  "type": "CALIBRATION",
  "params": {
    "type": "PIECEWISE",
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
```

Or isotonic:
```json
{
  "id": "calibrate",
  "type": "CALIBRATION",
  "params": {
    "type": "ISOTONIC",
    "runtime_path": "calibration/vertical_isotonic_v2.json"
  }
}
```

---

### Output Config

```json
"output_config": {
  "emit_trace": true,
  "max_feed_rows": 20
}
```

`emit_trace: true` enables Contract 5 (Trace Event) — per-step score snapshots for all components.
Cost: ~2x Iguazu event volume. Use in treatment arms; leave `false` in control.

---

### Horizontal Pipeline Contract (Phase 2)

Same schema, applied to store/item ordering within each carousel. Key addition: `row_overrides`
lets specific carousels opt in/out of steps that don't apply globally.

**Horizontal step types:**

| Step type | What it does | Replaces |
|---|---|---|
| `MODEL_SCORING` | Score stores/items via Sibyl | `StoreCollectionScorer.regressionContext()` |
| `MULTIPLIER_BOOST` | Strategic boost (NV, DashPass, campaign) | `getScoreModifierMap()` inside scorer |
| `BUSINESS_RULES_SORT` | Open/closed, priority biz, campaign pin rules | `modifyLiteStoreCollection()` when-chain |
| `ORDER_HISTORY_RERANK` | S1/S2/S3 tier reranking by order history | `FavoritesCarousel.rerankDecoratedEntities()` |

```json
{
  "control": {
    "p_act": {
      "predictor_name": "FEED_RANKING_SIBYL_PREDICTOR_NAME",
      "model_name": "store_ranker_v1",
      "features": [
        { "name": "STORE_ID",         "source": "STORE" },
        { "name": "CONSUMER_ID",      "source": "CONSUMER" },
        { "name": "STORE_ETA",        "source": "STORE" },
        { "name": "DAY_OF_WEEK",      "source": "CONTEXT" },
        { "name": "IS_DASHPASS_USER", "source": "CONSUMER" }
      ],
      "calibration": { "type": "NONE" }
    },
    "v_act": { "gov": 1.0, "fiv": 0.0, "strategic": 0.0 },
    "horizontal_pipeline": {
      "steps": [
        { "id": "score",        "type": "MODEL_SCORING",       "params": { "predictor_ref": "p_act" } },
        { "id": "boost",        "type": "MULTIPLIER_BOOST",    "params": { "vertical_boost_weights": { "10": 1.2 } } },
        { "id": "rules",        "type": "BUSINESS_RULES_SORT", "params": { "apply_open_closed": true } },
        { "id": "order_rerank", "type": "ORDER_HISTORY_RERANK","params": { "enabled": false } }
      ],
      "row_overrides": {
        "FAVORITES": {
          "order_rerank": { "enabled": true, "tiers": ["S1", "S2", "S3"], "lookback_days": 10 }
        },
        "PICKUP_MAP": {
          "score": { "skip": true },
          "rules": { "sort_by": "DISTANCE" }
        },
        "BOOKMARKS": {
          "score": { "skip": true },
          "rules": { "sort_by": "SAVELIST_ORDER" }
        }
      }
    },
    "output_config": {
      "emit_trace": false
    }
  }
}
```

`row_overrides` semantics: engine applies default config, then shallow-merges the carousel-
specific override for that carousel type. `skip: true` on a step skips it for that carousel only.

`ORDER_HISTORY_RERANK` is notable: today it runs silently *after* Iguazu fires (S1/S2/S3
decoration), so logged positions ≠ positions users see. As a declared step it is traced, and
logged positions finally match actual positions.

---

### Comprehensive Experiment Cases

#### Case 1: New primary model version

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

#### Case 2: Adjust a vertical boost weight (NV)

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

Multiple treatments in one file. DV assigns users to a treatment name.

---

#### Case 3: New intent model

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

---

#### Case 4: Enable diversity reranking

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

---

#### Case 5: UCB exploration for cold-starting a new vertical

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

Adds an exploration bonus to under-exposed NV carousels. Gradual ramp-up without pinning.
No hardcoded "pin to position 2 for all users for 2 weeks."

---

#### Case 6: Strategic value function weights (Phase 3)

Test whether weighting FIV (first-impression value) improves NV trial without GOV regression.

```json
{
  "treatment_nv_fiv_strategic": {
    "extends": "control",
    "v_act": {
      "gov": 0.8,
      "fiv": 0.15,
      "strategic": 0.05
    }
  }
}
```

---

#### Case 7: Cohort-specific config

Different boost for new users (low order frequency) vs high-frequency users.

```json
{
  "treatment_cohort_aware": {
    "extends": "control",
    "cohort_overrides": {
      "low_frequency_users": {
        "step_params": {
          "blend": {
            "vertical_boost_weights": {
              "boosting_multipliers": [{ "vertical_ids": [10], "multiplier": 1.8 }],
              "default_multiplier": 1.0
            }
          }
        }
      },
      "high_frequency_users": {
        "step_params": {
          "blend": {
            "vertical_boost_weights": {
              "boosting_multipliers": [{ "vertical_ids": [10], "multiplier": 1.1 }],
              "default_multiplier": 1.0
            }
          }
        }
      }
    }
  }
}
```

`cohort_overrides` resolved from `ConsumerContext` fields (e.g. `l28d_order_frequency_cohort`).
Same merge semantics as `step_params` — cohort override shallow-merges on top of base treatment.

---

#### Case 8: Removing a legacy heuristic (measurement-gated removal)

First, make the heuristic explicit and traced. Then remove it in a treatment.

```json
{
  "treatment_no_nv_pin": {
    "extends": "control",
    "step_params": {
      "pin": {
        "rules": [
          { "row_id": "pad",            "position": 3, "hard_pin": true },
          { "row_id": "member_pricing", "position": 0, "hard_pin": true }
        ]
      }
    }
  }
}
```

NV post-checkout pin removed from `rules`. Engine routes NV carousel on score alone. Trace events
show `score_before` vs `score_after` for NV at each step — opportunity cost is measurable.

---

#### Case 9: New pipeline step (requires BE to register processor first)

```json
{
  "treatment_with_merch_boost": {
    "vertical_pipeline": {
      "steps": [
        { "id": "score",     "type": "MODEL_SCORING",    "params": { "predictor_ref": "p_act" } },
        { "id": "blend",     "type": "MULTIPLIER_BOOST", "params": { ... } },
        { "id": "diversity", "type": "DIVERSITY_RERANK", "params": { "enabled": true, "weight": 0.4 } },
        { "id": "merch",     "type": "MERCH_PLACEMENT",  "params": { "campaign_boost": 1.3 } },
        { "id": "pin",       "type": "FIXED_PINNING",    "params": { ... } }
      ]
    }
  }
}
```

`MERCH_PLACEMENT` is a new processor type. Merch Platform team implements it, HP registers it
once. From that point any experiment can use it via config with no further BE involvement.

---

### Decision Guide

| What you're changing | Mode | Case |
|---|---|---|
| New model version | Mode 1 — override `p_act` | Case 1 |
| Adjust boost weight | Mode 1 — override `step_params.blend` | Case 2 |
| New intent model + calibration | Mode 1 — override `p_act` + `step_params.blend` | Case 3 |
| Enable diversity | Mode 1 — override `step_params.diversity` | Case 4 |
| Cold-start via UCB | Mode 1 — override `step_params.score` | Case 5 |
| Strategic value weights | Mode 1 — override `v_act` | Case 6 |
| Cohort-specific params | Mode 1 — add `cohort_overrides` | Case 7 |
| Remove a legacy heuristic | Mode 1 — remove from `step_params.pin.rules` | Case 8 |
| New step type | Mode 2 — full `vertical_pipeline` (BE registers processor first) | Case 9 |

---

### What MLEs Never Touch

- Step execution order — hardcoded by the engine (you cannot score before you have a model, boost before scores, diversity-rerank before boost, pin before final scores)
- Sibyl RPC, retries, timeouts — BE owns
- Feature fetching implementation — BE owns (you declare names and sources, BE fetches)
- `pImp` position decay — BE owns
- Trace schema and Kafka logging — BE owns, auto-emitted after every step
- Processor registry — BE owns (you use registered step types, BE adds new ones)
