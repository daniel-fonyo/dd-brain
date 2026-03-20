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

1. **Per-layer experiment traffic management without DV waterfall.** An MLE declares an experiment
   (layer + model + traffic %) and UBP allocates traffic within that layer. No dummy DVs, no
   priority lists, no reserve segments, no manual waterfall configuration. This is the #1 pain
   point and the #1 deliverable.

2. **Vertical ranking is config-driven.** The engine reads an internal `control.json` maintained
   by HP. Shadow comparison shows it replicates prod at >99% sort-order match.

3. **Horizontal ranking is config-driven.** Same engine architecture applied to store-within-carousel
   ranking.

4. **Simple MLE experiment interface.** An MLE provides: experiment name, layer, model name,
   traffic %. No step pipeline authoring. No HP code change.

Together: traffic allocation is declarative, the engine is generic, and MLEs self-serve experiments.

---

## Non-Goals

| Out of scope | Reason |
|---|---|
| Ads blending (ad candidates competing with organic stores) | Post-POC — requires shared scoring scale and unified value function |
| Multi-vertical cross-scoring (Rx vs NV in one score space) | Requires isotonic calibration + shared value function — post-POC |
| Explicit `pImp` position decay | Steps are position-unaware during scoring |
| Explicit `vAct` weights (`gov_w`, `fiv_w`, `strategic_w`) | `MODEL_SCORING` approximates vAct in POC via blending; explicit config is post-POC |
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

### Contracts at a Glance

| # | Contract | Facing | Counterparty | Problem solved |
|---|---|---|---|---|
| 0 | **Experiment Declaration** | MLE-facing | MLEs, partner teams | DV waterfall hell — per-layer traffic management |
| 1 | Experiment Config (JSON) | Internal | HP team | Config fragmented across 10+ locations |
| 2 | FeedRow / RowItem interfaces | Internal | Any team whose content gets ranked | 9+ typed classes with no shared interface |
| 3 | Value Function / Score | Internal | Model teams | Incomparable scoring scales across content types |
| 4 | RankingStep interface | Internal | Domain teams (NV, Ads, Merch) | No extension point without HP involvement |
| 5 | Trace Event | Both | Analytics / data science | No stage-wise score observability |

**Key distinction:** Contract 0 is the MLE-facing surface — what MLEs interact with. Contracts
1–4 are internal engine architecture that HP owns and maintains. MLEs never author pipeline
configs or step sequences.

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

## Contract 0: Experiment Declaration (MLE-facing)

**Counterparty: MLEs and partner teams**

This is the only contract MLEs interact with. An MLE declares an experiment — UBP handles
traffic allocation, engine configuration, and execution.

### What an MLE provides

```json
{
  "experiment_id": "nv_boost_v2",
  "layer": "vertical",
  "model_name": "vertical_intent_model_v3",
  "predictor_name": "FEED_RANKING_FW_SIBYL_PREDICTOR_NAME",
  "traffic_pct": 5,
  "params": {
    "vertical_boost_weights": {
      "boosting_multipliers": [{ "vertical_ids": [10], "multiplier": 1.5 }]
    }
  }
}
```

| Field | Required | Description |
|---|---|---|
| `experiment_id` | Yes | Unique experiment name |
| `layer` | Yes | `"vertical"` or `"horizontal"` |
| `model_name` | Yes | Sibyl model to use |
| `predictor_name` | No | Defaults to layer's default predictor |
| `traffic_pct` | Yes | Percentage of traffic for this experiment (within the layer) |
| `params` | No | Step-level param overrides (optional — empty = model swap only) |

### What UBP handles (MLE never touches)

- **Traffic allocation**: UBP assigns users to experiments within a layer. No dummy DVs, no
  priority lists, no reserve segments, no waterfall.
- **Pipeline execution**: UBP reads internal `control.json`, applies MLE overrides, runs the
  engine.
- **Control protection**: Control config is HP-owned. MLEs cannot modify it. Separate runtime
  configs for treatment params.
- **Tracing**: Automatic per-step score snapshots when enabled.

### Per-layer traffic management

**The current pain (DV waterfall):**
```
4 consumer cohorts → priority list → dummy DV for traffic splitting
→ reserve segment → waterfall to next experiment
→ change one experiment in the middle → all downstream experiments break
```

**UBP target state:**
```
MLE says: "I want 5% traffic on vertical layer for my experiment"
UBP allocates it. Done.

Two vertical experiments + one horizontal experiment running simultaneously?
Vertical experiments are mutually exclusive within vertical layer.
Horizontal experiment is orthogonal — no interference.
```

### N × M experiment model

Experiments are mutually exclusive WITHIN a layer, orthogonal ACROSS layers.

```
Layer: vertical    → Experiment A (5%), Experiment B (10%), Control (85%)
Layer: horizontal  → Experiment C (3%), Control (97%)
→ Any (vertical, horizontal) combination is valid
```

UBP owns the traffic router per layer. Each user gets exactly one assignment per layer.

---

## Contract 1: Engine Config (Internal — HP-owned)

**Counterparty: HP team only (not MLE-facing)**

Internal JSON defining the ranking pipeline. HP maintains `control.json` as the canonical
production behavior. MLE experiment declarations are resolved against this config at runtime.

### File layout

```
ubp/config/control.json           ← HP owns; canonical production behavior
ubp/config/overrides/             ← Generated from MLE experiment declarations
```

Config hot-reloaded via `DeserializedRuntimeCache` (5-min TTL). No pod restart.

### Resolution flow

```
1. UBP traffic router assigns user to experiment per layer
2. Engine reads control.json (base pipeline)
3. Engine applies MLE experiment overrides (model name, params)
4. Engine executes resolved pipeline
```

### Vertical step types (Phase 1)

Two coarse steps. We are NOT boiling the ocean — finer decomposition is a future iteration
once interfaces are proven.

| Type | What it wraps | Replaces today |
|---|---|---|
| `MODEL_SCORING` | `getScoreBundle()` — Sibyl gRPC + `BlendingUtil.blendBundle()` including diversity rerank (all one atomic call) | `EntityRankerConfiguration.getScoreBundleWithWorkflowHelper()` + `BlendingUtil` |
| `BOOST_AND_RANK` | `getBoostBundle()` + `getRankingBundle()` + `getRankableContent()` — score assignment to domain objects, position boosting, deal multiplier, pin vs flow sort order, reassembly (all one atomic flow) | `Boosting.kt` + `BoostingBundle.boosted()` + `RankingBundle.ranked()` |

### Horizontal step types (Phase 1.5)

Two coarse steps, mirroring the vertical decomposition.

| Type | What it wraps | Replaces today |
|---|---|---|
| `MODEL_SCORING` | `scoredCollections()` — Sibyl + score modifiers (atomic) | `StoreCollectionScorer` |
| `RANKING_SORT` | `modifyLiteStoreCollection()` `when(rankingType)` dispatch | `DefaultHomePageStoreRanker.modifyLiteStoreCollection()` when-chain |

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
score = MODEL_SCORING  →  BOOST_AND_RANK
      ≈    pAct(c)     ×      vAct(c)

pImp = 1.0 (not yet — steps are position-unaware during scoring)
```

Step order is load-bearing: `MODEL_SCORING` sets the base score (Sibyl + blending + diversity);
`BOOST_AND_RANK` handles score assignment, position boosting, deal multiplier, pin vs flow sort.
Calibration types available per experiment: `NONE` (within-type ranking), `PIECEWISE`
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

One event per (FeedRow, step) pair. 20 rows × 2 steps = 40 events per request. Enable in
treatment arms only; leave off in control to manage volume.

**What this enables:**
- Stage-wise score snapshots queryable in Snowflake without code changes
- Opportunity cost of pinning: `score_before` at `BOOST_AND_RANK` = organic value displaced
- Validate model changes actually move scores (not silently swallowed by calibration)
- Measurement-gated heuristic removal: measure impact, then remove

---

## Phases

### Phase 0 — Per-Layer Traffic Management (NEW — #1 Priority)

**The #1 pain point.** Replaces the DV waterfall with declarative per-layer experiment allocation.

**What to build:**
1. **Experiment registry** — runtime JSON listing active experiments per layer, each with
   `experiment_id`, `model_name`, `traffic_pct`, `params` overrides.
2. **Traffic router** — given a request, assigns user to exactly one experiment per layer based
   on declared traffic percentages. Deterministic (hash-based on consumer ID + layer).
3. **Control protection** — control config is a separate, team-owned runtime JSON. Treatment
   configs are in a separate namespace. No cross-contamination.

**Design constraint:** Must work with or without the ranking engine. Phase 0 can route traffic
to experiments that still use the old ranking code path — the engine is not a prerequisite.

**Open questions (to resolve before April 13 — YZ pat leave):**
- UBP-owned composite DV per layer vs. traffic router reading experiment registry?
- How does this interact with the existing 4-cohort system?
- Gradual migration: can old experiments coexist with UBP-managed experiments?

### Phase 1 — Vertical Ranking Engine

**Incision point:** `DefaultHomePagePostProcessor.reOrderGlobalEntitiesV2()`, replacing
`rankAndMergeContent()` for requests routed by Phase 0's traffic router.

**Migration (5 stages):**
1. **Contract assembly** — `UbpContractAssembler` runs in shadow mode, observes prod per-request
   decisions, assembles equivalent UBP contract JSON, logs it for visual inspection. Engine not
   required yet — the assembler runs independently. Fast path to seeing the contract shape.
2. **Freeze + shadow validate** — once contract shape stabilizes across 1000+ requests, freeze a
   representative assembled contract into static `control.json`. Shadow with static file. Target:
   `divergence_count = 0`.
3. **Control arm** — small % real traffic to UBP control. Identical result to old path.
4. **First experiment** — MLE declares experiment via Contract 0. No code change. This is the payoff.
5. **Retire** — as old experiments end, remove old code paths one by one.

**Stays unchanged:** All `NonRankableHomepageOrderingUtil` fixups (NV post-checkout pin, PAD=3,
member pricing), color bleed reordering, immersive spacing.

### Phase 1.5 — Horizontal Ranking Engine

**Incision point:** `DefaultHomePageStoreRanker.rank()`, replacing each
`modifyLiteStoreCollection()` call.

Same 5-stage migration path as Phase 1.

Phase 0 + Phase 1 + Phase 1.5 together prove the full claim: traffic allocation is declarative,
both ranking layers are config-driven, and MLEs self-serve experiments.

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
          "id": "boost_and_rank",
          "type": "BOOST_AND_RANK",
          "params": {
            "boost_by_position_enabled": false,
            "deal_carousel_score_multiplier": 1.0,
            "boost_by_position_allow_list": []
          }
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
        { "id": "score", "type": "MODEL_SCORING", "params": { "predictor_ref": "p_act" } },
        { "id": "rank",  "type": "RANKING_SORT",  "params": { "ranking_type": "CAROUSEL", "apply_open_closed": true } }
      ],
      "row_overrides": {
        "FAVORITES":   { "rank": { "ranking_type": "CAROUSEL", "order_history_rerank_enabled": true } },
        "PICKUP_MAP":  { "score": { "skip": true }, "rank": { "ranking_type": "PICKUP_STORE_FEED", "sort_by": "DISTANCE" } },
        "BOOKMARKS":   { "score": { "skip": true }, "rank": { "ranking_type": "BOOKMARKS", "sort_by": "SAVELIST_ORDER" } },
        "WHOLE_ORDER": { "rank": { "ranking_type": "WHOLE_ORDER_REORDER", "order_history_rerank_enabled": true } },
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

## Appendix B: MLE Experiment Examples (Contract 0)

These show what MLEs write — the simplified experiment declaration, not internal pipeline configs.

### New model version (most common)

```json
{
  "experiment_id": "vertical_fw_v4",
  "layer": "vertical",
  "model_name": "store_ranker_fw_v4",
  "predictor_name": "FEED_RANKING_FW_SIBYL_PREDICTOR_NAME",
  "traffic_pct": 5
}
```

### Adjust NV boost weight

```json
{
  "experiment_id": "nv_boost_1_5x",
  "layer": "vertical",
  "model_name": "store_ranking_v1_1",
  "traffic_pct": 3,
  "params": {
    "vertical_boost_weights": {
      "boosting_multipliers": [{ "vertical_ids": [10], "multiplier": 1.5 }],
      "default_multiplier": 1.0
    }
  }
}
```

### Multiple treatments in one experiment

```json
{
  "experiment_id": "nv_boost_sweep",
  "layer": "vertical",
  "model_name": "store_ranking_v1_1",
  "treatments": [
    { "name": "nv_1_5x", "traffic_pct": 2, "params": { "vertical_boost_weights": { "boosting_multipliers": [{ "vertical_ids": [10], "multiplier": 1.5 }] } } },
    { "name": "nv_2x",   "traffic_pct": 2, "params": { "vertical_boost_weights": { "boosting_multipliers": [{ "vertical_ids": [10], "multiplier": 2.0 }] } } }
  ]
}
```

### Enable diversity reranking

```json
{
  "experiment_id": "diversity_v1",
  "layer": "vertical",
  "model_name": "store_ranking_v1_1",
  "traffic_pct": 5,
  "params": {
    "diversity": { "enabled": true, "weight": 0.4 }
  }
}
```

### Partner team experiment (NV gives model + predictor)

```json
{
  "experiment_id": "nv_split_predictor_v1",
  "layer": "vertical",
  "model_name": "nv_vertical_model_v1",
  "predictor_name": "NV_SPLIT_PREDICTOR",
  "traffic_pct": 3
}
```

Note: NV team provides model + predictor. UBP allocates traffic. No DV waterfall setup required.

---

## Appendix C: Internal Engine Config Examples (Contract 1 — HP-owned)

These are maintained by HP, not authored by MLEs. Included for reference.

See `context/mle-experiment-guide.md` for the full internal config format and all legacy cases.
