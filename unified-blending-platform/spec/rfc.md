# RFC: Unified Blending Platform (UBP) — POC

> **POC RFC.** This document covers the proof-of-concept scope only: vertical ranking (Phase 1)
> and horizontal ranking (Phase 1.5). Ads blending, multi-vertical cross-scoring, explicit
> position decay, and value function weight config are all post-POC.

---

## Context and Abbreviations

| Term | Meaning |
|---|---|
| **UBP** | Unified Blending Platform — this system |
| **HP** | Homepage |
| **DV** | Dynamic Variable — DoorDash feature flag / experiment assignment |
| **UR** | Universal Ranker — Sibyl-based model for vertical carousel ranking |
| **SR** | Store Ranker — ranking within each carousel (horizontal) |
| **Sibyl** | DoorDash ML inference platform |
| **MLE** | Machine Learning Engineer |
| **NV** | New Verticals (grocery, convenience, alcohol, pets) |
| **Rx** | Restaurants |
| **GOV** | Gross Order Value |
| **FIV** | First Impression Value — value of a user's first order in a new vertical |
| **PAD** | Perks and Deals (Taste of DashPass carousel) |
| **pAct** | P(user takes action) — CTR, CVR, or order probability from ML model |
| **vAct** | Value of that action — weighted GOV + FIV + strategic |
| **pImp** | P(user sees a given page position) — position decay |
| **FeedRow** | A vertical ranking unit — one carousel on the page |
| **RowItem** | A horizontal ranking unit — one store or item within a carousel |
| **control.json** | JSON file canonically defining production ranking behavior |

---

## Problem Statement

The DoorDash homepage started as a single-vertical product (just Restaurants). Over time it grew to serve multiple content types on the same page: Rx stores, NV (grocery/convenience) stores, item carousels, ads, deals, DashPass merchandising. Each of these was built independently by different teams with their own ranking logic.

The result: **there is no single system deciding what goes where on the page**. Instead there are
4–5 separate systems each doing their own thing, stitched together with heuristics:

- Organic stores ranked by one Sibyl model
- Ads inserted after organic ranking by a separate blender
- NV stores boosted by hardcoded multipliers
- Merchandising carousels pinned by campaign rules
- Post-ranking fixups (NV position, PAD position, member pricing) that silently override everything before them

Nobody can answer: "Is this NV carousel worth more to the user than this Rx carousel at position 3?" — because the scores are on completely different scales and computed by different systems.

**Engineering consequence:** Every new experiment requires 10–15 file changes, 2+ weeks of HP
engineer involvement, and logic that accumulates permanently. MLEs cannot ship experiments without HP. Partner teams cannot add custom logic without modifying the core pipeline.

---

## Goals

**POC must prove:**

1. **Vertical ranking is config-driven.** An MLE drops a JSON file — no HP code change, no pod
   restart. Shadow comparison shows `control.json` replicates prod at >99% sort-order match.

2. **Horizontal ranking is config-driven.** An MLE can run a store-within-carousel ranking
   experiment by dropping a JSON file, the same way as vertical.

Together: the engine is generic, experiments are config. Both ranking layers proven.

---

## Non-Goals

| Out of scope | Reason |
|---|---|
| Ads blending (ad candidates competing with organic stores) | Post-POC — requires shared scoring scale and unified value function |
| Multi-vertical cross-scoring (Rx vs NV in one score space) | Requires isotonic calibration + shared value function — post-POC |
| Explicit `pImp` position decay | Steps are position-unaware during scoring |
| Explicit `vAct` weights (`gov_w`, `fiv_w`, `strategic_w`) | `MULTIPLIER_BOOST` approximates vAct in POC; explicit config is post-POC |
| Partner-owned custom steps | Engine must prove stable first |
| Post-ranking business rule fixups (NV post-checkout pin, PAD=3, member pricing) | Business constraints, not MLE experiments — stay in `NonRankableHomepageOrderingUtil` |
| Color bleed reordering, immersive content spacing | Presentation constraints, not ranking |

---

## High Level Design

### Two Ranking Layers

```
REQUEST
  │
  ▼  Layer 1: Retrieval
  │  DefaultHomePageStoreContentFetching.fetch()
  │
  ▼  Layer 2: Grouping
  │  Buckets candidates into named carousels
  │
  ▼  Layer 3: Horizontal Ranking  ← Phase 1.5  (RowItemRanker)
  │  Ranks stores/items WITHIN each carousel
  │
  ▼  Layer 4: Vertical Ranking    ← Phase 1    (FeedRowRanker)
  │  Ranks carousels for their vertical page position
  │
  ▼  Post-processing (unchanged)
     NonRankableHomepageOrderingUtil (NV pin, PAD, member pricing, color bleed, spacing)
```

### Five Contracts at a Glance

| # | Contract | Counterparty | Problem solved |
|---|---|---|---|
| 1 | Experiment Config (JSON) | MLEs, partner teams | Config fragmented across 10+ locations |
| 2 | FeedRow / RowItem interfaces | Any team whose content gets ranked | 9+ typed classes with no shared interface |
| 3 | Value Function / Score | Model teams | Incomparable scoring scales across content types |
| 4 | RankingStep interface | Domain teams (NV, Ads, Merch) | No extension point without HP involvement |
| 5 | Trace Event | Analytics / data science | No stage-wise score observability |

### Engine Architecture

```
FeedRowRanker.rank(rows, experimentId, treatment, context)
  │
  ├── 1. UbpRuntimeUtil.resolve(experimentId, treatment)
  │        → ResolvedPipeline(steps, predictors, emitTrace)
  │        → unknown treatment? → fall back to "control" (Null Object)
  │        → "extends: control"? → shallow-merge params + predictors (Prototype)
  │
  ├── 2. for each StepConfig:
  │        step = stepRegistry[config.type] ?: skip + warn
  │        resolvedParams = resolveParams(config.rawParams, predictors)
  │        typedParams = OBJECT_MAPPER_RELAX.treeToValue(resolvedParams, step.paramsClass)
  │        snapshot scores (if tracing)
  │        step.process(rows, context, typedParams)     ← Chain of Responsibility
  │        traceEmitter?.recordStep(...)                ← Observer (when emit_trace=true)
  │
  └── 3. return rows sorted by score descending
```

`RowItemRanker` (Phase 1.5) is the same architecture applied to `MutableList<RowItem>`.

---

## Contract 1: Experiment Config (JSON)

**Counterparty: MLEs and partner teams**

One JSON file per experiment. One DV per experiment. No HP code change to run a new experiment.

### File layout

```
ubp/experiments/control.json           ← HP owns; all treatments inherit from this
ubp/experiments/{experiment_id}.json   ← MLE owns
```

DV label: `ubp_{experiment_id}`. DV value: treatment name (e.g. `"treatment_fw_v4"`).
Unknown treatment → engine silently uses `"control"`.

```kotlin
// One line in DiscoveryExperimentManager.Manifest:
UBP_HP_VERTICAL_FW_V4("ubp_hp_vertical_fw_v4"),
```

Config hot-reloaded via `DeserializedRuntimeCache` (5-min TTL). No pod restart.

### Two modes

**Mode 1 — Param override** (most experiments)

Inherit the full control pipeline; change one or more step params.

```json
{
  "treatment_fw_v4": {
    "extends": "control",
    "step_params": {
      "score": { "model_name": "store_ranker_fw_v4" }
    }
  }
}
```

Merge: each key in `step_params[stepId]` shallow-merges into that step's params. Treatment wins
on collision. Keys not mentioned are preserved from control.

**Mode 2 — Full pipeline declaration** (step sequence changes)

```json
{
  "treatment_diversity_v1": {
    "vertical_pipeline": {
      "steps": [
        { "id": "score",     "type": "MODEL_SCORING",    "params": { ... } },
        { "id": "blend",     "type": "MULTIPLIER_BOOST", "params": { ... } },
        { "id": "diversity", "type": "DIVERSITY_RERANK", "params": { "enabled": true, "diversity_scoring_params": { "weight": 0.4 } } },
        { "id": "pin",       "type": "FIXED_PINNING",    "params": { ... } }
      ]
    }
  }
}
```

### N × M experiment model

Horizontal and vertical DVs are independent. A user is assigned to one of each simultaneously.

```
DV "ubp_hp_horizontal_v2" → "treatment_fw"       (horizontal)
DV "ubp_hp_vertical_v3"   → "intent_model_v3"    (vertical)
→ N × M valid cells, no interference between layers
```

### Vertical step types (Phase 1)

| Type | What it does | Replaces today |
|---|---|---|
| `MODEL_SCORING` | Sibyl scoring — assigns `pAct` to each FeedRow | `EntityRankerConfiguration.getScoreBundleWithWorkflowHelper()` |
| `MULTIPLIER_BOOST` | Calibration × intent × vertical boost weights (`≈ vAct`) | `BlendingUtil.blendBundle()` |
| `DIVERSITY_RERANK` | Greedy rerank penalizing non-Rx density | `BlendingUtil.rerankEntitiesWithDiversity()` |
| `POSITION_BOOSTING` | Deal carousel multiplier + boost-by-position enforcement + NV unpinning | `Boosting.kt` logic after blending |
| `FIXED_PINNING` | MLE-configured position pins (not the hardcoded business fixups) | `BoostingBundle.boosted()` |

### Horizontal step types (Phase 1.5)

| Type | What it does | Replaces today |
|---|---|---|
| `MODEL_SCORING` | Score organic + ad stores together via Sibyl | `StoreCollectionScorer.regressionContext()` |
| `SCORE_MODIFIER` | Per-business/store score multipliers from runtime JSON | `getScoreModifierMap()` in `StoreCollectionScorer.kt` |
| `CAMPAIGN_SORT` | Campaign-positioned stores + business sort order overrides | `StoreCarouselService.sortStoreEntitiesForCarousels()` |
| `BUSINESS_RULES_SORT` | Open/closed sort + AlwaysOpen type + priority business/vertical IDs | `StoreCarouselService` comparator chain + `StoreStatusComparator` |
| `ORDER_HISTORY_RERANK` | S1/S2/S3 reranking by order history | `WholeOrderReorderFilterUtil.getGoToOrders()` |

---

## Contract 2: FeedRow / RowItem Interfaces

**Counterparty: Any team whose content gets ranked**

The engine never sees typed carousel classes — only these interfaces. Adapters are the only code
aware of both sides.

```kotlin
interface FeedRow {
    val id: String
    val type: RowType                          // enum — one entry per carousel domain type
    var score: Double                          // mutable — steps write in-place
    val metadata: MutableMap<String, Any>      // side-channel for inter-step data
    fun applyBackTo()                          // writes final score back to domain object
}

interface RowItem {
    val id: String
    val type: RowItemType
    var score: Double
    val metadata: MutableMap<String, Any>
    fun applyBackTo()
}
```

Each domain type gets one adapter:

```kotlin
class StoreCarouselRow(private val carousel: StoreCarousel) : FeedRow {
    override val id    = carousel.id
    override val type  = RowType.STORE_CAROUSEL
    override var score = carousel.predictionScore ?: 0.0
    override val metadata = mutableMapOf<String, Any>()
    override fun applyBackTo() { carousel.sortOrder = score.toInt() }
}
fun StoreCarousel.toFeedRow(): FeedRow = StoreCarouselRow(this)
```

There is no `AD_CAROUSEL` FeedRow. Ads are post-POC scope.

### RowType enum

```kotlin
enum class RowType {
    STORE_CAROUSEL, ITEM_CAROUSEL, DEAL_CAROUSEL,
    STORE_COLLECTION, COLLECTION_V2, ITEM_COLLECTION,
    MAP_CAROUSEL, REELS_CAROUSEL, STORE_ENTITY,
}
```

No `NV_CAROUSEL` — NV stores appear as `STORE_CAROUSEL` or `ITEM_CAROUSEL`. NV-specific step
logic uses carousel ID matching in params, not a distinct RowType.

---

## Contract 3: Value Function

**Counterparty: Model teams (Universal Ranker, Ads, VP ranker)**

```
EV(c, k) = pImp(k) × pAct(c) × vAct(c)

  pImp(k)  = P(user sees position k) — position decay, BE-owned, post-POC
  pAct(c)  = P(user acts | they see c) — ML model output
  vAct(c)  = Value of that action — gov_w × GOV + fiv_w × FIV + strategic_w × Strategic
```

**POC approximation:**

```
score = MODEL_SCORING  ×  MULTIPLIER_BOOST
      ≈    pAct(c)     ×      vAct(c)

pImp = 1.0 (not yet — steps are position-unaware during scoring)
```

Step order is load-bearing: `MODEL_SCORING` sets the base score; `MULTIPLIER_BOOST` multiplies
into it. Calibration types available per experiment: `NONE` (within-type ranking), `PIECEWISE`
(score-range multipliers), `ISOTONIC` (regression table — post-POC for cross-type scoring).

---

## Contract 4: RankingStep Interface

**Counterparty: Domain teams that need custom logic**

HP owns the engine and registry. Partner teams implement and own their steps.

```kotlin
interface FeedRowRankingStep {
    val stepType: String                         // key in experiment config "type" field
    val paramsClass: Class<out StepParams>       // engine uses this to deserialize rawParams

    suspend fun process(
        rows: MutableList<FeedRow>,
        context: RankingContext,
        params: StepParams,                      // typed — deserialized by engine before this call
    )
}
```

`params` is `StepParams` (sealed interface), not `Map<String, Any>`. The engine calls
`OBJECT_MAPPER_RELAX.treeToValue(rawParams, step.paramsClass)` before each `step.process()`.
Steps do a single safe cast — guaranteed correct because the engine used `paramsClass`
to deserialize.

**The `params` constraint is absolute.** A step that reads its own DV keys internally is not a
UBP step — it defeats the purpose. All tunable behavior flows through `params`.

Adding a step type:
1. Team implements `FeedRowRankingStep`
2. HP adds one line to `buildStepRegistry()`
3. Any experiment can use it via config — no further HP involvement

`RowItemRankingStep` is the identical interface for horizontal ranking.

---

## Contract 5: Trace Event

**Counterparty: Analytics and data science**

Emitted automatically after every step when `output_config.emit_trace: true`. Steps have zero
tracing code — the engine observes and emits (Observer pattern).

```protobuf
// services-protobuf/ubp_feed_row_ranking_trace.proto
message UbpFeedRowRankingTrace {
  string request_id    = 1;
  string experiment_id = 2;
  string treatment     = 3;
  string step_id       = 4;
  string row_id        = 5;
  string row_type      = 6;
  double score_before  = 7;
  double score_after   = 8;
  int64  timestamp_ms  = 9;
}
```

One event per (FeedRow, step) pair. 20 rows × 3 steps = 60 events per request. Enable in
treatment arms only; leave off in control to manage volume.

**What this enables:**
- Stage-wise score snapshots queryable in Snowflake without code changes
- Opportunity cost of pinning: `score_before` at `FIXED_PINNING` = organic value displaced
- Validate model changes actually move scores (not silently swallowed by calibration)
- Measurement-gated heuristic removal: measure impact, then remove

---

## Phases

### Phase 1 — Vertical Ranking

**Incision point:** `DefaultHomePagePostProcessor.reOrderGlobalEntitiesV2()`, replacing
`rankAndMergeContent()` for requests with a UBP DV.

**Migration (5 stages):**
1. **Contract assembly** — `UbpContractAssembler` runs in shadow mode, observes prod per-request
   decisions, assembles equivalent UBP contract JSON, logs it for visual inspection. Engine not
   required yet — the assembler runs independently. Fast path to seeing the contract shape.
2. **Freeze + shadow validate** — once contract shape stabilizes across 1000+ requests, freeze a
   representative assembled contract into static `control.json`. Shadow with static file. Target:
   `divergence_count = 0`.
3. **Control arm** — small % real traffic to UBP control. Identical result to old path.
4. **First experiment** — MLE drops a JSON file. No code change. This is the payoff.
5. **Retire** — as old experiments end, remove old code paths one by one.

**Stays unchanged:** All `NonRankableHomepageOrderingUtil` fixups (NV post-checkout pin, PAD=3,
member pricing), color bleed reordering, immersive spacing.

### Phase 1.5 — Horizontal Ranking

**Incision point:** `DefaultHomePageStoreRanker.rank()`, replacing each
`modifyLiteStoreCollection()` call.

Same 5-stage migration path as Phase 1.

Phase 1 + Phase 1.5 together prove the core claim: both ranking layers are config-driven.

---

## Appendix A: Control JSON Baselines

### Vertical control.json

```json
{
  "control": {
    "predictors": {
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
      }
    },
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
            "calibration_config":     { "calibration_entries": [], "default_multiplier": 1.0 },
            "intent_scoring_config":  { "feature_configs": [], "score_lookup_entries": [], "default_score": 1.0 },
            "vertical_boost_weights": {
              "boosting_multipliers": [], "default_multiplier": 1.0,
              "item_carousel_boosting_multipliers": [], "item_carousel_default_multiplier": 1.0
            }
          }
        },
        {
          "id": "diversity",
          "type": "DIVERSITY_RERANK",
          "params": { "enabled": false, "diversity_scoring_params": { "weight": 0.0 } }
        },
        {
          "id": "pin",
          "type": "FIXED_PINNING",
          "params": { "rules": [] }
        }
      ]
    },
    "output_config": { "emit_trace": false, "max_feed_rows": 20 }
  }
}
```

### Horizontal control.json

```json
{
  "control": {
    "predictors": {
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
      }
    },
    "horizontal_pipeline": {
      "steps": [
        { "id": "score",        "type": "MODEL_SCORING",        "params": { "predictor_ref": "p_act" } },
        { "id": "boost",        "type": "MULTIPLIER_BOOST",     "params": { "vertical_boost_weights": { "boosting_multipliers": [], "default_multiplier": 1.0 } } },
        { "id": "rules",        "type": "BUSINESS_RULES_SORT",  "params": { "apply_open_closed": true } },
        { "id": "order_rerank", "type": "ORDER_HISTORY_RERANK", "params": { "enabled": false } }
      ],
      "row_overrides": {
        "FAVORITES":   { "order_rerank": { "enabled": true, "tiers": ["S1","S2","S3"], "lookback_days": 10 } },
        "PICKUP_MAP":  { "score": { "skip": true }, "rules": { "sort_by": "DISTANCE" } },
        "BOOKMARKS":   { "score": { "skip": true }, "rules": { "sort_by": "SAVELIST_ORDER" } },
        "WHOLE_ORDER": { "order_rerank": { "enabled": true } },
        "NOOP":        { "steps": [] }
      }
    },
    "output_config": { "emit_trace": false }
  }
}
```

`row_overrides` semantics: default config applies to all carousels; the carousel-specific override
shallow-merges on top. `"skip": true` on a step skips it for that carousel only.

---

## Appendix B: Experiment Examples

See `context/mle-experiment-guide.md` for the full decision guide and all experiment cases.

### New model version

```json
{
  "treatment_fw_v4": {
    "extends": "control",
    "predictors": {
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
}
```

### Adjust NV boost weight

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
  }
}
```

### Enable diversity reranking

```json
{
  "treatment_diversity_v1": {
    "extends": "control",
    "step_params": {
      "diversity": {
        "enabled": true,
        "diversity_scoring_params": {
          "weight": 0.4,
          "coefficient_for_non_rx": 0.6,
          "local_window_size_for_non_rx": 3,
          "local_coefficient_for_non_rx": 0.3
        }
      }
    }
  }
}
```

### Measure opportunity cost of a pin

Remove a pin in a treatment. Trace events show `score_before` at the `FIXED_PINNING` step —
the organic value that was being displaced becomes measurable for the first time.

```json
{
  "treatment_no_pad_pin": {
    "extends": "control",
    "step_params": {
      "pin": { "rules": [] }
    }
  }
}
```

### New intent model with calibration

```json
{
  "treatment_intent_v3": {
    "extends": "control",
    "predictors": {
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
