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

### Two Distinct Score Fields
There are TWO separate intent-related scores in ScoreWithFactors:

| Field | Key | What it is |
|-------|-----|------------|
| `userIntentPredictionScore` | `user_intent_prediction_score` | Older TLDR user intent score (CollectionScorer, uses `FEED_RANKING_FW_SIBYL_PREDICTOR_NAME` predictor with `TLDR_USER_INTENT_PREDICTOR_NAME` model) |
| `verticalIntentPredictionScore` | `vertical_intent_multiplier` (via `verticalIntentMultiplier`) | **This is Michael's new model** (BaseEntityScorer, uses `VERTICAL_INTENT_PREDICTION` predictor, model ID from `VerticalBlendingConfig.intentPredictionModel`) |

### Logging Status
- `user_intent_prediction_score` → **Already logged** in `score_modifiers` (ContainerEventsGenerator line 200, 2305-2315)
- `vertical_intent_multiplier` → **Already logged** in `score_modifiers` (ContainerEventsGenerator line 202, 2335-2345)
- **Raw `verticalIntentPredictionScore`** → Logged via `StoreCarouselDataAdapterUtil` and `ItemCarouselDataAdapter` to facet logging, BUT **not as a separate score_modifier in Iguazu cross-vertical events**

### Key Code Paths
- Predictor registration: `SibylRegressor.kt:276-289` (gated by `enableVerticalIntentPrediction` experiment)
- Score extraction: `SibylRegressor.kt:498,523` (verticalIntentPredictions)
- Blending application: `VerticalBlending.kt:304-314` (multiplies entity score)
- Score propagation: `ScoreBundle.kt:256-257` → `ScoreWithFactors.verticalIntentPredictionScore`
- Iguazu event logging: `ContainerEventsGenerator.kt` (score_modifiers list)

## What Daniel Needs To Do

### The Ask
Michael confirmed with Frank that the intent predict model score is **not yet logged**. He needs the raw prediction score added to the `score_modifier` column in cross-vertical logging.

### Analysis: What's Missing
The **raw prediction score** from the intent model (`verticalIntentPredictionScore`) is partially plumbed:
- It exists in `ScoreWithFactors` (line 21)
- It flows through `VerticalBlending` where it's used in scoring
- It's logged in carousel-level facet logging

**But**: In the Iguazu `score_modifiers` list (ContainerEventsGenerator), only the **multiplier** (`vertical_intent_multiplier`) is logged — not the raw prediction probability from the model.

### Implementation Plan
Add a new `score_modifier` entry for the raw intent prediction score:

1. **DomainUtilLoggingConstants.kt** — Add constant (if not reusing `USER_INTENT_PREDICTION_SCORE_KEY`, may need new key like `VERTICAL_INTENT_PREDICTION_SCORE_KEY`)
2. **ContainerEventsGenerator.kt** — Add new `LoggedValue` enum entry + add to `containerScoreModifierValues` list
3. **Verify** the score is already populated in the logging map from `ScoreWithFactors.verticalIntentPredictionScore`

**Effort estimate**: Low — follows exact same pattern as every other score_modifier. Copy-paste from existing entries.

**No dependency on Michael's side** — this is purely feed-service logging plumbing.

## Future Work
- V2: Business-aware modeling (store-type features, embeddings)
- Hierarchical taxonomy (subtypes within restaurant/NV)
- Sequence/recency signals (attention over recent events)
- Exploration-aware routing (diversify on high-entropy predictions)
- Expand back to multi-class when data supports it

## References
- Model PR: https://github.com/doordash/dd-models/pull/4491
- Fabricator training data: https://github.com/doordash/fabricator/tree/master/fabricator/repository/features/cx_discovery/intent_prediction
- Meeting notes: https://notes.granola.ai/t/62c5fdf5-34fe-4ce6-bc23-c239f3d4dc61
- One Pager: [HP] Intent Prediction Model V1
