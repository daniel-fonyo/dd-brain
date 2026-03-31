# Intent Prediction Model V1

## Overview
ML model to predict user category-level intent on the DoorDash homepage: "what type of store is this user most likely to order from right now?"

**Team**: Michael Chen, Yu Zhang, Zhenzhen Liu (Eng), Chunlei Li (EM), Parul Khurana (PM)

## Model Architecture
- **Type**: DNN with softmax head (MLP: Linear → LayerNorm → GELU → Dropout)
- **Loss**: Multinomial log loss (CrossEntropyLoss)
- **Training**: Positive samples only (conversion events/orders)
- **Output**: Probability distribution over intent classes

### V0 (Current)
- **Binary classification**: Restaurant (class 0) vs Non-Restaurant (class 1)
- Collapsed from original 6-class (restaurant, grocery, retail, alcohol, drugstore, other) due to extreme class imbalance (~89% restaurant)
- **Model ID**: `1774285074`
- **Offline metrics**: 91.6% accuracy, but NV recall only 4.2% (precision 20.9%)

### V1.1 (New features added)
- Added: `last_order_business_vertical_id`, `hours_since_last_same_vertical_purchase`, `is_weekend`, `proportion_non_restaurant_conversions_l28d`, `avg_store_views_per_session`, `avg_clicks_per_session`
- **Offline metrics**: 86.8% accuracy (lower but more balanced), NV recall improved to 35.9% (precision 74.4%), macro AUC 0.908

## Class Imbalance Lessons Learned
1. Never stack sampler + focal loss + class weights — they multiply each other
2. Focal loss + class weights compound — use `focal_gamma=0.0`
3. Balanced sampler kills calibration (5.3× train/inference mismatch)
4. Loss weights need data to work — rare classes need to actually appear in batches
5. **Weighted sampler is the right lever** — partial reweighting, class-0 stays dominant
6. Merging rare classes reduces problem difficulty

## Feed-Service Integration

### Original Integration PR (merged)
**PR #58350**: `hp_intent_pred` by Michael Chen + Zhenzhen Liu — merged 2026-02-18
- Branch: `hp_intent_pred`
- 15 files changed: +121/-4

#### What PR #58350 did (end-to-end model integration):
1. **PredictionEnums.kt** — Added `VERTICAL_INTENT_PREDICTION("vertical_intent_prediction")` predictor name
2. **VerticalBlendingConfig.kt** — Added `intentPredictionModel` field (JSON: `intent_prediction_model`) for model ID from runtime config
3. **P13nExperimentManager.kt** — Added `enableVerticalIntentPrediction()` gating: returns true when `HP_VERTICAL_BLENDING_CONFIG` experiment starts with `"intent_model__"`
4. **SibylRegressor.kt** — Registered the vertical intent predictor (gated by experiment + config model ID), extracted predictions from Sibyl response into `verticalIntentPredictions` map, added to `ScoresMetadata`
5. **BaseEntityScorer.kt** — Wired `VERTICAL_INTENT_PREDICTION` predictor name into `AdditionalPredictors`, extracted per-entity vertical intent scores async, added to `AdditionalPrediction.verticalIntentPrediction`
6. **ScoreBundle.kt** — Propagated `verticalIntentPredictionScore` from `AdditionalPrediction` into `ScoreWithFactors` in all 9 score bundle branches
7. **ScoreWithFactors.kt** — Added `var verticalIntentPredictionScore: Double? = null` field
8. **VerticalBlending.kt** — Core scoring logic: if `config.intentPredictionModel.isNotEmpty()` AND `verticalIntentPredictionScore != null`, multiply entity score by `verticalIntentPredictionScore * intentMultiplier`; else just `intentMultiplier`. Also propagated score through `BlendableEntity`.

#### Key design decisions from PR #58350:
- Score is **per-entity** (each carousel gets its own prediction score based on `business_vertical_id`)
- Model is called via **Sibyl multi-predictor** — piggybacks on the existing prediction request with other predictors
- Gated by **experiment prefix** (`intent_model__`) — config ID must start with this prefix for model to be active
- Score **multiplies** the intent multiplier from config (not replaces it): `entityScore *= verticalIntentPredictionScore * intentMultiplier`

### Two Distinct Score Fields
There are TWO separate intent-related scores in ScoreWithFactors:

| Field | Key | What it is |
|-------|-----|------------|
| `userIntentPredictionScore` | `user_intent_prediction_score` | Older TLDR user intent score (CollectionScorer, uses `FEED_RANKING_FW_SIBYL_PREDICTOR_NAME` predictor with `TLDR_USER_INTENT_PREDICTOR_NAME` model) |
| `verticalIntentPredictionScore` | `vertical_intent_prediction_score` | **Michael's new model** (BaseEntityScorer, uses `VERTICAL_INTENT_PREDICTION` predictor, model ID from `VerticalBlendingConfig.intentPredictionModel`) |

### Logging Status (after our PR)
- `user_intent_prediction_score` → **Logged** in `score_modifiers`
- `vertical_intent_multiplier` → **Logged** in `score_modifiers`
- `vertical_intent_prediction_score` → **Now logged** in `score_modifiers` (our PR `feat/intent-prediction-score-logging`)

### Key Code Paths
- Predictor name: `PredictionEnums.kt` — `VERTICAL_INTENT_PREDICTION("vertical_intent_prediction")`
- Experiment gating: `P13nExperimentManager.enableVerticalIntentPrediction()` — config must start with `"intent_model__"`
- Model ID source: `VerticalBlendingConfig.intentPredictionModel` (runtime JSON config)
- Predictor registration: `SibylRegressor.kt:227-242` (gated by experiment + config)
- Score extraction: `SibylRegressor.kt:332-335,347,372` (verticalIntentPredictions from Sibyl response)
- Per-entity mapping: `BaseEntityScorer.kt:227-231` → `AdditionalPrediction.verticalIntentPrediction`
- Score bundle propagation: `ScoreBundle.kt` (9 branches all map `verticalIntentPredictionScore`)
- Blending application: `VerticalBlending.kt:280-291` — `entityScore *= verticalIntentPredictionScore * intentMultiplier`
- Logging: `ContainerEventsGenerator.kt` — `VERTICAL_INTENT_PREDICTION_SCORE` in score_modifiers

## Our Work: Score Logging PR

### The Ask
Michael confirmed with Frank that the intent predict model score is **not yet logged** in Iguazu events. He needs the raw prediction score added to the `score_modifier` column in cross-vertical logging.

### What We Built
**PR**: `feat/intent-prediction-score-logging` — 12 files (8 prod + 4 test)
- Added `VERTICAL_INTENT_PREDICTION_SCORE_KEY = "vertical_intent_prediction_score"` constant
- Added `verticalIntentPredictionScore` field to `StoreCarousel` and `ItemCarousel` domain models
- Mapped field through `EntityRankerConfiguration` (4 copy blocks)
- Added to logging adapters: `StoreCarouselDataAdapterUtil`, `ItemCarouselDataAdapter`, `CategoryPreviewUniversalItemCarouselMapper`
- Added `LoggedValue.VERTICAL_INTENT_PREDICTION_SCORE` enum entry in `ContainerEventsGenerator`
- Full test coverage in 4 test files

**No scoring logic changes** — pure logging plumbing. Zero dependency on Michael's side.

## Future Work
- V2: Business-aware modeling (store-type features, embeddings)
- Hierarchical taxonomy (subtypes within restaurant/NV)
- Sequence/recency signals (attention over recent events)
- Exploration-aware routing (diversify on high-entropy predictions)
- Expand back to multi-class when data supports it

## References
- Model integration PR: https://github.com/doordash/feed-service/pull/58350
- Model PR: https://github.com/doordash/dd-models/pull/4491
- Fabricator training data: https://github.com/doordash/fabricator/tree/master/fabricator/repository/features/cx_discovery/intent_prediction
- Meeting notes: https://notes.granola.ai/t/62c5fdf5-34fe-4ce6-bc23-c239f3d4dc61
- One Pager: [HP] Intent Prediction Model V1
