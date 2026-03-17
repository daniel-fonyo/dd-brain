# How Experiments, Predictors, Models, and Ranking Work Today

> Ground truth reference. Describes the actual current system — not the aspiration.
> Use this to understand what UBP is replacing and why.

---

## Overview: The Cast of Characters

| Concept | What It Is | Where It Lives |
|---|---|---|
| **Sibyl** | DoorDash's internal ML prediction serving system | External RPC service |
| **Predictor** | A named slot in Sibyl that serves a specific model's outputs | `PredictorName` enum |
| **Model** | A specific trained artifact loaded into a predictor slot | `modelOverrideId` string |
| **Experiment** | A DV treatment + its associated config JSON | DV key + runtime JSON |
| **Score** | A composite number assembled from multiple predictor outputs | `ScoreWithFactors` |
| **Blending** | Post-Sibyl multipliers applied to adjust per-vertical/intent | `VerticalBlending.kt` |
| **Ranking** | Sorting carousels by their final score and assigning `sortOrder` | `EntityRankerConfiguration.kt` |

---

## 1. Sibyl: What It Is

Sibyl is DoorDash's internal ML prediction service. Feed-service sends it a batch of feature vectors (one per carousel or store) and Sibyl returns a prediction score for each one.

```
Feed-Service → SibylPredictionServiceRepository
                    │
                    ▼
              Sibyl (external RPC)
              ├─ Receives: List<PredictionFeatureSet> + List<Predictor>
              └─ Returns: PredictionResponse
                          └─ per-predictor, per-featureSet: { score: Double }
```

A single Sibyl request can include **multiple predictors at once** — the response bundles all results together. This is the multi-predictor call pattern.

**What Sibyl returns per prediction:**
- `BINARY_CLASSIFICATION_PROBABILITY`: a probability in [0, 1] — most common, used as pConv
- `REGRESSION_VALUE`: a real-valued output
- `EMBEDDING_OUTPUT`: a float array (used for intent model — takes index [2] for intent prediction)

---

## 2. Predictors: Named Model Slots

A **predictor** is an identifier that tells Sibyl which model family to use. A **model override** is the specific trained version within that family.

```kotlin
// PredictorName enum — the full list of predictor slots used today:
FEED_RANKING_FW_SIBYL_PREDICTOR_NAME           // Main ranking predictor (FW = FeedWord)
UNIVERSAL_RANKER_MULTIPLIER_SIBYL_PREDICTOR_NAME // Score multiplier predictor
PROGRAMMATIC_BOOSTING_PREDICTOR_NAME           // Programmatic boost adjustments
UCB_UNCERTAINTY_PREDICTOR_NAME                 // Exploration bonus (UCB)
MAB_PREDICTOR_NAME                             // Multi-armed bandit bonus
VERTICAL_INTENT_PREDICTION                     // Intent model for vertical blending
// + VP predictors:
feedRankingDasherCostPredictorName             // pDasherCost for VP experiments
feedRankingExpectedVpPredictorName             // pCommission/expected VP
```

**Role of each predictor in the final score:**

| Predictor | Role | How Combined |
|---|---|---|
| Main (FW) | Base conversion probability — the core pAct signal | Multiplicative base |
| Multiplier | Per-store/carousel scaling factor (e.g. SR boost) | `× multiplierScore` |
| Programmatic Boost | Business-rule-based boost for campaigns | `× progBoostScore` |
| UCB Uncertainty | Exploration bonus for under-explored stores | `+ ucbScore` |
| MAB | Multi-armed bandit bonus | `+ mabScore` |
| Vertical Intent | Intent signal used in blending (not in SibylRegressor formula) | Applied later in BlendingUtil |
| VP (DasherCost + Commission) | For VP-mode scoring: replaces base with profit estimate | Replaces base formula |

---

## 3. How an Experiment Selects Predictors and Models

There are **two independent wiring mechanisms** for selecting which predictor/model runs for a given user:

### Mechanism A: DV → predictor name switch

Some predictors are turned on/off via DV boolean logic:

```kotlin
// Example: use FW predictor for store ranker?
fun shouldUseSRFWPredictorForStoreRanker(experimentMap): Boolean {
    return getExperimentValue(experimentMap, "enable_store_ranker_continuous_trained")
        .endsWith("_feed_ranking_fw")
}

// Example: UCB exploration enabled?
fun shouldUcbUncertaintyForStoreRanker(experimentMap, rankingType): Boolean {
    return !isControl(experimentMap, "discovery_p13n_exploration_ranker_for_store_ranker")
}
```

These checks are **scattered across `P13nExperimentManager.kt`** — one per feature, no unified registry.

### Mechanism B: DV → model override ID via runtime JSON lookup

The main predictor's **specific model version** is resolved via a runtime JSON map:

```
DV "enable_store_ranker_continuous_trained" = "treatment_fw_v2"
          │
          ▼
P13nRuntimeUtil.getDVTreatmentSibylModelMappingConfig()
    reads: ranking/dv_treatment_sibyl_model_mapping.json
    {
      "FEED_RANKING_FW_SIBYL_PREDICTOR_NAME": {
        "treatment_fw_v2": "store_ranker_fw_v2_model_id",
        "treatment_fw_v3": "store_ranker_fw_v3_model_id",
        ...
      },
      "UNIVERSAL_RANKER_MULTIPLIER_SIBYL_PREDICTOR_NAME": {
        "treatment_fw_v2": "multiplier_model_v2",
        ...
      }
    }
          │
          ▼
modelName = "store_ranker_fw_v2_model_id"  ← sent as modelOverrideId to Sibyl
```

This means the experiment DV value doesn't directly contain a model name — it's a key into a JSON lookup table. The JSON file is hot-reloadable via `DeserializedRuntimeCache` (5-minute TTL).

### Mechanism C: DV → blending config ID → VerticalBlendingConfig

For blending parameters (calibration, intent multipliers, boost weights), a separate DV and JSON are used:

```
DV "hp_vertical_blending_config" = "intent_model_v3"
          │
          ▼
P13nRuntimeUtil.getVerticalBlendingConfigMap()
    reads: ranking/hp_vertical_blending_config.json
    {
      "control": { calibration: {...}, intentModel: "", boostWeights: {...} },
      "intent_model_v3": { calibration: {...}, intentModel: "intent_v3_model_id", boostWeights: {...} },
      ...
    }
          │
          ▼
VerticalBlendingConfig for this user's experiment
```

**The intent model name is inside the blending config**, not in the model mapping JSON. This is a different config seam from the one used for the main predictor.

---

## 4. The Score Assembly Pipeline

For each carousel, the score is assembled across two stages:

### Stage 1: SibylRegressor — the raw multi-predictor score

```
Sibyl response → extracted per-predictor scores → combined:

rawScore         = sibylPredictions[featureSetId]           // main predictor output
multiplierScore  = multiplierPredictions[featureSetId]      // default 1.0 if missing
progBoostScore   = programmaticBoostPredictions[featureSetId] // default 1.0 if missing
ucbScore         = ucbUncertaintyPredictions[featureSetId]  // default 0.0 if missing
mabScore         = mabPredictions[featureSetId]             // default 0.0 if missing

sibylCompositeScore = (rawScore × multiplierScore × progBoostScore)
                    + ucbScore
                    + mabScore
```

This is stored in `ScoreWithFactors.finalScore` after Stage 1. All individual factors are also retained for logging.

**VP mode (if VP experiment active):**
The base `rawScore` is replaced by a predicted-profit calculation:
```
vpScore = pConv × pDasherCost × pCommission × alpha × beta
```
where `pConv` is from main predictor, `pDasherCost` and `pCommission` from their own Sibyl predictors, and `alpha`/`beta` are config params from `vp_ranker_treatment_config.json`.

### Stage 2: BlendingUtil — calibration × intent × boost

This stage runs only when `enableHomepageVerticalBlending` DV is active. It takes the `sibylCompositeScore` from Stage 1 and applies three multiplicative adjustments:

```
blendedScore = sibylCompositeScore
             × calibrationMultiplier(verticalId, score)     // piecewise range lookup
             × intentPredictionScore                        // from intent Sibyl predictor
             × intentScoringMultiplier(features)            // feature-lookup-table product
             × verticalBoostWeight(verticalId, bizVertId)   // static per-vertical multiplier
             + diversityBonus                               // if diversity reranking enabled
```

**Calibration** is a piecewise function: for a given `(verticalId, scoreBucket)` it returns a static multiplier. Used to normalize scores across verticals so they're comparable.

**Intent scoring** combines two sub-signals:
- `intentPredictionScore`: output of the intent Sibyl model (embedding index [2])
- `intentScoringMultiplier`: a feature lookup table — takes context features like `is_dashpass`, `day_of_week`, `nv_lifestage_cohort` → looks up a precomputed score, multiplies all matching entries together

**Vertical boost weight** is a static multiplier per `(verticalId, businessVerticalId, carouselIdPrefix)` — manually tuned, not learned. This is one of the 30+ heuristics UBP aims to replace.

### Stage 3: BoostingBundle — post-ranking fixups

After blending, a separate set of adjustments runs outside the scored pipeline:
- `enforceManualSortOrder()` — overrides sort position for pinned carousels
- `updateSortOrderOfNVCarousel()` — NV-specific position fixup
- `updateSortOrderOfPADCarousel()` — PAD carousel fixup
- Gap rules, member pricing fixups

These run **after** all scoring and have no tracing or observability. They're the most brittle part of the pipeline.

---

## 5. Features: What Goes Into Sibyl

Each item scored by Sibyl gets a `PredictionFeatureSet` — a `Map<String, FeatureValue>` containing signals about the store/carousel and the user context.

**Context features** (per request, same for all carousels):
```
local_day_hour, day_of_week
is_dashpass, is_occasional_user
nv_lifestage_cohort, l28d_order_frequency_cohort, order_count_l28d
is_post_checkout
```

**Entity features** (per carousel):
```
vertical_id, business_vertical_id, carousel_id_prefix
store-level features (store id, cuisine type, rating, delivery time, etc.)
realtime signals (if enabled via DV): recent order history, click signals
promo/campaign features (if enabled via DV)
```

**Features for the intent model** are the same context features — the intent model predicts user intent toward a vertical (RX vs. NV, etc.) based on user cohort signals.

---

## 6. Experiment Anatomy: What a Real Experiment Looks Like

A typical new experiment today involves:

**1. New model trained offline** → deployed to Sibyl under a new model ID (e.g. `store_ranker_fw_v4`)

**2. New DV treatment value added** (e.g. `treatment_fw_v4`) → this is the experiment handle

**3. Model mapping JSON updated:**
```json
// ranking/dv_treatment_sibyl_model_mapping.json
{
  "FEED_RANKING_FW_SIBYL_PREDICTOR_NAME": {
    "treatment_fw_v4": "store_ranker_fw_v4"
  }
}
```

**4. Optionally: blending config updated** if the experiment also changes calibration/intent/boost:
```json
// ranking/hp_vertical_blending_config.json
{
  "treatment_fw_v4": {
    "calibration_config": { ... },
    "intent_prediction_model": "",
    "vertical_boost_weights": { ... }
  }
}
```

**5. DV traffic split configured** — users in `treatment_fw_v4` get the new model.

**Why this is expensive today:**
- DV setup is manual and requires coordinating two independent DV keys (one for model, one for blending)
- The model mapping JSON and blending config JSON are separate files with separate ownership
- The intent model name lives inside the blending config, not in the model mapping — easy to forget
- VP predictor params live in yet another JSON (`vp_ranker_treatment_config.json`)
- Each boolean flag (UCB, MAB, programmatic boost) is a separate DV check in `P13nExperimentManager`
- No single place where "this experiment uses these predictors with these params" is declared

---

## 7. How the Vertical Score Drives Ranking

After scoring, `EntityRankerConfiguration` walks through all carousel types and copies `ScoreWithFactors.finalScore` → `carousel.predictionScore`. Then:

```
All carousels (store carousels, item carousels, collections, NV, etc.)
    → sorted descending by predictionScore
    → sortOrder integers assigned: 0, 1, 2, ...
    → pinned carousels skip this: their sortOrder is forced by PinnedCarouselUtil
    → post-ranking fixups adjust NV, PAD, gap positions
    → output: HomePageStoreLayoutOutputElements with final sortOrder
```

**Key architectural fact:** All 9 carousel types are flattened into a single pool before scoring. `EntityRankerConfiguration.getEntities()` does this:

```kotlin
return listOf(
    listOfNotNull(content.dealCarousel).map(::ScorableEntityDealCarousel),
    content.storeCarousels.map(::ScorableEntityStoreCarousel),
    content.storeCollections.map(::ScorableEntityStoreCollection),
    content.collectionsV2.map(::ScorableEntityCollectionV2),
    content.itemCarousels.map(::ScorableEntityItemCarousel),
    content.itemCollections.map(::ScorableEntityItemCollection),
    if (shouldRerankStoreList) content.storeList.stores.map(::ScorableEntityStoreEntity) else emptyList(),
    content.mapCarousels.map(::ScorableEntityMapCarousel),
    listOfNotNull(content.reelsCarousel).map(::ScorableEntityReelsCarousel),
).flatten()
```

They're all scored together by Sibyl, but the 9 types have **no shared interface** — they're only unified inside `getEntities()` via the `ScorableEntity` wrapper. After scoring, they're immediately separated back into typed collections. This is the core structural problem UBP fixes with `VerticalComponent`.

---

## 8. The Current "Value Function" vs. the Aspirational One

**What the northstar says:**
```
Expected Value = pImp × pAct × vAct
```

**What the code actually computes:**
```
Stage 1:  sibylScore = (rawSibyl × multiplier × progBoost) + ucb + mab
Stage 2:  finalScore = sibylScore × calibration × intentPred × intentLookup × boostWeight
```

The gap:
- `pAct` is approximated by `rawSibyl` (a conversion probability from the FW predictor), but it's not labeled or separated as such in the code
- `vAct` **does not exist** as a unified concept today. VP experiments approximate it via `pConv × pDasherCost × pCommission × alpha × beta`, but this only applies to the FW predictor path and isn't unified with organic/NV/merch ranking
- `pImp` (position-decay impression probability) **is not modeled at all** today — positions are assigned after scoring, not used to influence the score
- `multiplier`, `progBoost`, `calibration`, `intentLookup`, `boostWeight` are all multiplicative adjustments that encode various business intent (boost NV, penalize low-quality stores, etc.) but none of them are declared as `vAct` components — they're scattered heuristics
- UCB and MAB bonuses are additive, making the multiplicative structure of the formula inconsistent

**In short:** today's scoring is a collection of manually assembled multipliers and additive bonuses that approximate a value function without being one. UBP's job is to make it an explicit, measurable, calibrated value function.

---

## 9. Key Files Reference

| What | File |
|---|---|
| Predictor names | `libraries/sdk-p13n/.../scoring/SibylRegressor.kt` |
| Multi-predictor assembly + score formula | `SibylRegressor.kt:290–501` |
| Experiment → predictor/model resolution | `P13nExperimentManager.kt`, `P13nRuntimeUtil.kt` |
| DV manifest | `P13nExperimentManager.Manifest` enum |
| Model mapping JSON | `ranking/dv_treatment_sibyl_model_mapping.json` |
| Blending config JSON | `ranking/hp_vertical_blending_config.json` |
| Blending logic (calibration × intent × boost) | `VerticalBlending.kt:292–338` |
| Score bundle (all factors) | `ScoreWithFactors.kt` |
| Carousel type flattening | `EntityRankerConfiguration.kt:getEntities()` |
| sortOrder assignment | `EntityRankerConfiguration.kt:getHomepageScoredItemCarousels()` |
| Post-ranking fixups (no tracing) | `NonRankableHomepageOrderingUtil.kt`, `DefaultHomePagePostProcessor.kt` |
| Pinning config | `carousels/pinned_carousel_ranking_order.json` |
| VP ranker config | `ranking/vp_ranker_treatment_config.json` |
| Runtime cache | `DeserializedRuntimeCache.kt` (5-min TTL) |
