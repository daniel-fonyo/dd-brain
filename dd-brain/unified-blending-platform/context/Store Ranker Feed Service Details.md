# Overview

**What is the store ranker?** The store ranker is an Sibyl ML model that assigns a prediction scores (pConv) to rank each entities horizontally within carousels, as well as the first page (up to 50) of store feed.

# Implementation

In homepage pipeline, the set of store carousels are ranked in the **homepageHOMRankerJob**. In this job we also rank cuisine filters, campaign carousels, and item carousels. We have separate ML models for to rank each of these different types of entities.

For store carousels, the rank function of the **DefaultHomePageStoreRanker** class is used which calls **StoreCollectionScorer**. Below are details regarding this collection scorer.

## Purpose

The scorer enables personalized store ranking by:

- Computing ML-based prediction scores for stores using Sibyl models
    
- Applying strategic business multipliers and score modifiers
    
- Supporting multiple ranking strategies (regression, multi-label classification)
    
- Generating features from real-time store data and consumer context
    
- Handling different page types and ranking contexts
    

## Architecture

### Dependencies

The class depends on four main components injected via constructor:

1. SibylRegressor: Computes regression-based prediction scores
    
2. SibylMultiLabels: Computes multi-label classification scores
    
3. PercentageMatchCacheRepository: Caches percentage match scores for stores
    
4. P13nExperimentManager: Manages personalization experiments and feature flags
    

### Core Data Flow

`Collections + Stores + Context         ↓ [Group by RankingType]         ↓ [Create Feature Sets] ← Consumer context, geo data, store attributes         ↓ [Compute Scores] ← Sibyl ML models (regression or multi-label)         ↓ [Apply Modifiers] ← Business multipliers, boosts         ↓ [Build ScoreWithFactors] ← Final scores, multipliers, percentage match         ↓ Scored Collections`

## Main API

### scoreCollections()

**Signature:**

`suspend fun scoreCollections(     context: ScorerContext,     stores: List<DiscoveryStore>,     collections: List<LiteStoreCollection>, ): List<LiteStoreCollection>`

**Purpose:** Returns a copy of input collections with storePredictionScoresMap populated with computed scores.

**Parameters:**

- context: Contains consumer, geo, device, and experiment context
    
- stores: List of available stores to score
    
- collections: Store collections to be scored (each may have different ranking types)
    

**Returns:** Collections with populated score maps. If scoring fails, score maps will be empty.

**Key Steps:**

1. **Group stores by ranking context** (lines 93-120)
    
    - Creates RegressionContext for each unique ranking type
        
    - Groups stores that share the same model configuration
        
    - Filters out collections with RankingType.NOOP
        
2. **Compute scores in parallel** (lines 136-157)
    
    - Uses supervisorScope and async for concurrent scoring
        
    - Calls computeScoresForConfig() for each context group
        
    - Stores results in scoresMetadataByConfig
        
3. **Calculate percentage match** (lines 158)
    
    - Derives percentile-based match scores for eligible pages
        
4. **Build final collections** (lines 160-226)
    
    - Maps scores back to collections
        
    - Creates ScoreWithFactors for each store containing:
        
        - finalScore: Main prediction score
            
        - multiLabelScores: Individual label predictions (if applicable)
            
        - srMultiplier: Store ranker multiplier
            
        - percentageMatch: Consumer-store match percentage
            
        - srUncertaintyScore: Exploration/uncertainty score
            
        - rawScorerScores: Pre-boost scores
            
        - scoreBoostModifier: Applied boost amount
            

## Scoring Strategies

### Regression Scoring

**When used:** Most ranking scenarios (default)

**Method:** computeRegressionScores() (lines 329-379)

**Process:**

1. Calls SibylRegressor.computeRegressionScoresWithMultiPredictors()
    
2. Applies optional score boosting via business/store ID multipliers
    
3. Coerces final scores to MAX_SCORE (1.0)
    
4. Returns metadata including multipliers, percentage match, and uncertainty scores
    

**Additional Predictors:**

- **Multiplier predictor**: Adjusts scores based on business objectives
    
- **Percentage match predictor**: Consumer-store affinity
    
- **UCB uncertainty predictor**: Exploration for under-served stores
    

### Multi-Label Scoring

**When used:** When model name is in P13nRuntimeUtil.sibylMultiLabelsModelNames()

**Method:** computeMultiLabelScores() (lines 408-444)

**Process:**

1. Creates MultiLabelsContext from scorer context
    
2. Calls SibylMultiLabels.computeMultiLabelScores()
    
3. Extracts final scores and individual label values
    
4. Applies business multipliers
    
5. Returns scores with label breakdowns
    

**Use case:** Predicting multiple engagement metrics simultaneously (e.g., click, conversion, revenue)

## Feature Engineering

### createPredictionFeatureSet()

**Location:** Lines 475-870

**Purpose:** Transforms raw context and store data into ML model features.

**Feature Categories:**

#### 1. Entity IDs (Categorical)

- Experience ID
    
- Consumer ID
    
- Store ID, Business ID, Business Vertical ID
    
- Submarket ID, District ID
    
- Session ID, Device ID
    

#### 2. Temporal Features

- Day of week (PySpark format: 1-7)
    
- Hour of day (local time)
    
- Day part regroup (morning/afternoon/evening/night)
    
- Weekend indicator
    

#### 3. Store Attributes

- Delivery fee (raw and adjusted)
    
- ETA (estimated time of arrival)
    
- Distance (meters and miles)
    
- Price range
    
- Offer badge presence
    
- Next closing time (seconds to closing)
    

#### 4. Consumer Features

- Is DashPass subscriber
    
- Is occasional user
    
- Platform (iOS, Android, Web)
    

#### 5. Personalization Features (when enabled)

- Cuisine affinity similarity
    
- Preference-based cold starter features
    
- Horizontal position (if enabled)
    

#### 6. Surface-Aware Features (VLP/Shopping Tabs)

- Container code (carousel type)
    
- Page vertical ID code
    
- Scene ID (v1 and v2)
    
- Various contextual list features
    

#### 7. Model-Specific Features

- MOR (Multi-Objective Ranking) weights
    
- Distance/subtotal/delivery fee weights
    
- Retention weight
    

### Feature Formats

- **Entity IDs**: Used for embedding lookups in models
    
- **Numerical Features**: Float values (delivery fee, distance, etc.)
    
- **Categorical Features**: String categorical values (when not using list features)
    
- **List Features**: Int64 lists for factorization machine models
    

## Score Modifiers

### Business/Store Boosting

**Method:** getScoreModifierMap() (lines 310-326)

**When active:** P13nExperimentManager.shouldBoostBusinessForStoreRanker() returns true

**Logic:**

1. Retrieves multiplier maps by business ID and store ID
    
2. For each store:
    
    - First checks store-specific multipliers
        
    - Falls back to business-level multipliers
        
3. Multipliers are applied in computeRegressionScores() and computeMultiLabelScores()
    

**Effect:** finalScore = rawScore × multiplier (capped at 1.0)

**Use case:** Strategic boosting of specific merchants or brands

### MOR (Multi-Objective Ranking) Weight

**Method:** getMORWeight() (lines 1167-1194)

**Purpose:** Balances multiple objectives (relevance, distance, fees, retention)

**Determination priority:**

1. Retention ranker experiment (if active)
    
2. PAD V1 M1 distance ranker (if active)
    
3. General distance ranker (if active)
    
4. Delivery fee ranker (homepage only)
    
5. Default: 0.0
    

**Application:** Added as numerical features for distance, delivery fee, and retention weights

## Model Selection

### getModelName()

**Location:** Lines 872-890

**Purpose:** Selects appropriate Sibyl model based on experiments and page type

**Selection logic:**

1. Check PAD V1 M1 experiment (specific distance ranker variant)
    
2. Fall back to model from experiment configuration
    
3. Uses DVTreatmentSibylModelConfig to map experiments to models
    

### Regression Context Creation

**Method:** regressionContext() (lines 892-1011)

**Key fields:**

- predictorName: Determines which Sibyl predictor to use
    
- modelOverrideName: Specific model version
    
- useCase: Defines prediction scenario (homepage, vertical page, etc.)
    
- additionalPredictors: Multiplier, percentage match, UCB predictors
    
- rankingType: Type of ranking being performed
    

**Predictor selection based on:**

- Ranking type (curated carousel, campaign, store feed)
    
- Page type (homepage, vertical, shopping tabs)
    
- Active experiments (FW models, surface-aware features, auto-trained models)
    

## Use Cases and Page Types

### Homepage (PageType.HOMEPAGE, PageType.DEFAULT_HOMEPAGE)

- Use case: USE_CASE_HOME_PAGE_HORIZONTAL_RANKING
    
- Supports: Reorder ranking, NV carousel, FW predictors
    
- Special handling: Delivery fee ranker, continuous trained models
    

### Vertical Landing Pages

- Use case: USE_CASE_VERTICAL_PAGE_HORIZONTAL_RANKING (or FW/surface-aware variants)
    
- Supports: Surface-aware features, factorization machine models
    
- Special handling: VLP-specific predictors
    

### Shopping Tabs

- Use case: USE_CASE_SHOPPING_TABS_HORIZONTAL_RANKING (or FW/surface-aware variants)
    
- Similar to vertical pages with tab-specific configurations
    

### Special Carousels

- **NV Curated Item**: USE_CASE_LCM_NEARBY_STORE_RANKING
    
- **Campaign**: May use DFY promo-specific predictor
    
- **Inflation Multiplier**: Uses specific inflation model
    

# Adding Predictors to StoreCollectionScorer

## Overview

This guide walks through adding a new predictor to StoreCollectionScorer, which is the main scoring engine for store ranking in the feed service. Predictors are machine learning models that score stores based on consumer context and store attributes.

**Location:** libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/ranking/scoring/StoreCollectionScorer.kt

## Predictor Types

### Main Predictor

The primary scoring model that produces the base store ranking scores. Only one main predictor is active per scoring context.

**Examples:**

- FEED_RANKING_SIBYL_PREDICTOR_NAME - Standard feed ranking
    
- FEED_RANKING_FW_SIBYL_PREDICTOR_NAME - Factorization machine variant
    
- FEED_RANKING_AUTO_TRAINED_SIBYL_PREDICTOR_NAME - Continuously trained model
    

### Additional Predictors

Auxiliary models that provide supplementary signals combined with the main predictor's output.

**Current Additional Predictors:**

1. **Multiplier Predictor** (STORE_RANKER_MULTIPLIER_SIBYL_PREDICTOR_NAME)
    
    - Adjusts scores based on business objectives
        
    - Output: Score multiplier per store
        
2. **Percentage Match Predictor** (PERCENTAGE_MATCH_PREDICTOR_NAME)
    
    - Consumer-store affinity scoring
        
    - Output: Match percentage per store
        
3. **UCB Uncertainty Predictor** (STORE_RANKER_EXPLORATION_PREDICTOR_NAME)
    
    - Exploration/exploitation balancing
        
    - Output: Uncertainty score for exploration
        
4. **Expected Value Predictors** (Dasher Cost, Expected VP)
    
    - Used for advanced optimization strategies
        
    - Output: Cost and value predictions
        

## Architecture Overview

> **Data Flow**
> 
> RegressionContext contains:
> 
> - predictorName - Main predictor
>     
> - modelOverrideName - Model version
>     
> - additionalPredictors - Contains all auxiliary predictors (multiplier, percentage match, UCB, and your new predictor)
>     
> 
> Flow:
> 
> 1. RegressionContext is created with all predictor configurations
>     
> 2. SibylRegressor.computeRegressionScoresWithMultiPredictors() calls all configured predictors and returns ScoresMetadata
>     
> 3. StoreCollectionScorer applies predictions from all sources (base scores, multipliers, affinity, exploration, custom predictors)
>     

## Step-by-Step Guide

### Step 1: Define Predictor Name

Add your predictor name to the PredictorName enum.

**File:** libraries/sdk-core/src/main/kotlin/com/doordash/consumer/feed/sdk/core/models/ranking/PredictorName.kt

**Action:** Add a new enum value with your predictor's label string.

### Step 2: Update AdditionalPredictors Data Class

Add fields for your predictor to the AdditionalPredictors class.

**File:** libraries/discovery-sdk/src/main/kotlin/com/doordash/consumer/discovery/sdk/p13n/scoring/AdditionalPredictors.kt

**Action:** Add two nullable fields:

- myNewPredictorName: PredictorName? - The predictor name enum value
    
- myNewModelName: String? - The model version string
    

### Step 3: Update RegressionContext Creation

Modify the regressionContext() method in StoreCollectionScorer to include your predictor.

**File:** libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/ranking/scoring/StoreCollectionScorer.kt:892

**Action:** In the AdditionalPredictors instantiation within the RegressionContext return statement:

- Set myNewPredictorName to your predictor enum value
    
- Set myNewModelName by looking up from modelMap using your predictor's label and calling getModelNameFromExperiment()
    

### Step 4: Configure Model Mapping

Add your predictor's model configuration to the dynamic value (DV) system.

**File:** libraries/discovery-p13n-utils/src/main/kotlin/com/doordash/consumer/feed/discoveryp13nUtils/runtimeutils/PersonalizationRuntimeUtil.kt

**Action:** In getDVTreatmentSibylModelMappingConfig(), add a mapping entry for your predictor:

- Use your predictor's label as the map key
    
- Create a DVTreatmentSibylModelConfig with default model version and experiment-to-model mappings
    
- Define which experiments control model selection and what models they map to
    

### Step 5: Update ScoresMetadata

If your predictor returns data that needs to be stored and passed to downstream components, update the ScoresMetadata internal class.

**File:** libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/ranking/scoring/StoreCollectionScorer.kt:1118

**Action:** Add a nullable field to store your predictor's output (typically Map<String, Double>?).

### Step 6: Update SibylRegressor Integration

The SibylRegressor class handles the actual predictor calls. You'll need to update it to call your new predictor.

**File:** libraries/discovery-sdk/src/main/kotlin/com/doordash/consumer/discovery/sdk/p13n/scoring/SibylRegressor.kt

**Action:** In computeRegressionScoresWithMultiPredictors():

- Check if your predictor name is configured in regressionContext.additionalPredictors
    
- If yes, call sibylClient.predict() with your predictor name, model name, feature sets, and use case
    
- Store the predictions in a variable
    
- Add your predictions to the returned ScoresMetadata
    

### Step 7: Apply Predictor Output in StoreCollectionScorer

Update computeRegressionScores() or the score building logic to use your predictor's output.

**File:** libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/ranking/scoring/StoreCollectionScorer.kt:329

**Action:** After receiving ScoresMetadata from the regressor:

- Extract your predictor's scores from the metadata
    
- Apply them to the base scores (multiply, add, or filter based on your use case)
    
- Return updated ScoresMetadata with your predictor's data included
    

### Step 8: Update ScoreWithFactors

If you want to expose your predictor's output to consumers of the scoring API, add it to ScoreWithFactors.

**File:** libraries/discovery-sdk/src/main/kotlin/com/doordash/consumer/discovery/sdk/p13n/model/ScoreWithFactors.kt

**Action:** Add a nullable field for your predictor's output (typically Double?).

Then update the collection building logic in scoreCollections():

**File:** libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/ranking/scoring/StoreCollectionScorer.kt:196

**Action:** In the ScoreWithFactors instantiation, add your predictor's score by looking it up from scoresMetadata by store ID.

### Step 9: Add Predictor Name Tracking

Update the predictor name tracking in addPredictorNames() and addModelNames():

**File:** libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/ranking/scoring/StoreCollectionScorer.kt:242

**Action:**

- In addPredictorNames(), add logic to extract your predictor name from scoresMetadata.additionalPredictors and add it to the set
    
- In addModelNames(), add logic to extract your model name from scoresMetadata.additionalPredictors and add it to the set
    

### Step 10: Add Feature Engineering (If Needed)

If your predictor requires new features, add them to createPredictionFeatureSet():

**File:** libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/ranking/scoring/StoreCollectionScorer.kt:475

**Action:** In the feature set builder:

- Check if your predictor is active in regressionContext.additionalPredictors
    
- If yes, add numerical/categorical/list features required by your model
    
- Use the appropriate feature builder methods (addNumericalFeatures, addCategoricalFeatures, addListFeatures)
    

---

## Experiment Configuration

### Create Feature Flag

Add a feature flag to control your predictor.

**Action:** In your experiment manifest, add a new Experiment constant for your predictor.

### Add Experiment Check

Create a helper method to check if your predictor should be enabled.

**Action:** Create a function that calls PlatformExperimentManager.isTreatment() with your experiment label.

### Conditionally Enable Predictor

Update regressionContext() to conditionally enable your predictor based on experiment.

**Action:** Use your experiment check function to conditionally set predictor name and model name in AdditionalPredictors.

---

## Testing

### Unit Tests

Create unit tests for your predictor integration.

**File:** libraries/domain-util/src/test/kotlin/com/doordash/consumer/feed/domainUtil/ranking/scoring/StoreCollectionScorerTest.kt

**Actions:**

- Mock the regressor to return test scores including your predictor's output
    
- Call scoreCollections() with test context
    
- Assert that your predictor's scores appear in the results
    
- Assert that your predictor name is tracked in storePredictionPredictors
    

### Integration Tests

Test end-to-end with real Sibyl integration.

**Actions:**

- Set up real context with experiments enabled
    
- Call scoring and verify predictor was invoked
    
- Verify output is present in final results
    

---

## Monitoring and Observability

### Add Logging

Add sample logging for your predictor using LoggerUtil.sampledInfoLog().

**What to log:**

- Predictor name and model name
    
- Number of stores scored
    
- Sample of prediction values
    
- Consumer ID and page type
    

### Add Metrics

Track predictor performance using MetricsUtil.

**Metrics to add:**

- Histogram of prediction distribution
    
- Counter for invocations with tags (model name, page type)
    
- Latency metrics if applicable
    

---

## Common Patterns

### Predictor as Score Multiplier

If your predictor outputs should multiply the base score:

- Extract multiplier from predictor scores (default to 1.0 if missing)
    
- Multiply base score by multiplier
    
- Cap at MAX_SCORE (1.0)
    

### Predictor as Score Boost

If your predictor outputs should be added to the base score:

- Extract boost value from predictor scores (default to 0.0 if missing)
    
- Add boost to base score
    
- Cap at MAX_SCORE (1.0)
    

### Predictor as Filter

If your predictor should filter stores:

- Get eligible store IDs where predictor score exceeds threshold
    
- Filter final scores to only include eligible stores
    

### Conditional Predictor Application

If your predictor should only apply to certain ranking types:

- Check rankingType or pageType in regressionContext()
    
- Only set predictor name/model if conditions match
    
- Otherwise set to null
    

---

## Checklist

**Implementation Checklist:**

- [ ] Added predictor name to PredictorName enum
    
- [ ] Updated AdditionalPredictors data class
    
- [ ] Modified regressionContext() method
    
- [ ] Configured model mapping in dynamic values
    
- [ ] Updated ScoresMetadata if needed
    
- [ ] Integrated with SibylRegressor
    
- [ ] Applied predictor output in scoring logic
    
- [ ] Updated ScoreWithFactors if exposing to API
    
- [ ] Added predictor/model name tracking
    
- [ ] Added feature engineering if needed
    
- [ ] Created feature flag and experiment
    
- [ ] Added unit tests
    
- [ ] Added integration tests
    
- [ ] Added logging and metrics
    
- [ ] Updated documentation
    
- [ ] Tested with various page types and ranking contexts
    

---

## Troubleshooting

> **ℹ️ Common Issues**

#### Predictor Not Being Called

**Check:**

- Is the experiment enabled in your test context?
    
- Is the model mapping configured correctly?
    
- Is the predictor name in the additionalPredictors object?
    
- Check logs for any errors in SibylRegressor
    

#### Scores Not Being Applied

**Check:**

- Is the predictor output being returned in ScoresMetadata?
    
- Is the score application logic correct (multiply vs add)?
    
- Are scores being capped by MAX_SCORE (1.0)?
    
- Check if stores have null/missing predictor scores
    

#### Features Not Reaching Model

**Check:**

- Are features being added to the correct PredictionFeatureSet?
    
- Is feature name matching what the model expects?
    
- Are features conditionally added based on experiment?
    
- Check Sibyl logs for feature validation errors
    

---

## File Locations Summary

> **Key Files to Modify**

**Core Predictor Configuration:**

- libraries/sdk-core/.../PredictorName.kt - Add predictor name enum
    
- libraries/discovery-sdk/.../AdditionalPredictors.kt - Add predictor fields
    
- libraries/discovery-p13n-utils/.../PersonalizationRuntimeUtil.kt - Add model mapping
    

**Scoring Integration:**

- libraries/domain-util/.../StoreCollectionScorer.kt - Main integration point
    
    - regressionContext() method (line 892)
        
    - computeRegressionScores() method (line 329)
        
    - scoreCollections() method (line 196)
        
    - addPredictorNames() method (line 242)
        
    - createPredictionFeatureSet() method (line 475)
        
- libraries/discovery-sdk/.../SibylRegressor.kt - Add predictor call
    
- libraries/discovery-sdk/.../ScoreWithFactors.kt - Expose to API
    

**Testing:**

- libraries/domain-util/.../StoreCollectionScorerTest.kt - Add unit tests