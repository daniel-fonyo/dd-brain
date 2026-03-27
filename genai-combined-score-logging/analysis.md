# GenAI Combined Score — Logging Gap Analysis

**Date**: 2026-03-25

## Problem

The `combinedScore` (`finalScore^alpha * similarity^beta`) used to rerank stores in GenAI carousels is **not logged in Iguazu**. Only the original `finalScore` is logged as `HORIZONTAL_ELEMENT_SCORE`. This means we can't analyze the actual reranking signal in Snowflake.

## Code Trace

### 1. Combined score computed then discarded
**`GeneratedRecommendationCarouselService.kt:172-186`**
```kotlin
val combinedScore = finalScore.pow(alpha) * similarity.pow(beta)  // line 180
store to combinedScore                                             // line 181
...
val combinedStores = reRankedStoreScoreMap.map { it.first }       // line 186 — scores dropped
```
`collection.copy(entities = combinedStores)` at line 206 returns reordered stores but `storePredictionScoresMap` is **unchanged** — still has original ranker scores.

### 2. Decoration sets predictionScore = finalScore (not combinedScore)
**`HomepageProgrammaticProductServiceUtil.kt:209-215`**
```kotlin
val rankInfo = collection.storePredictionScoresMap[productStore.id]
predictionScore = rankInfo?.finalScore   // original ranker score
```

### 3. Adapter writes predictionScore into logging map
**`StoreCarouselDataAdapter.kt:2465-2466`**
```kotlin
store.predictionScore?.let { map[STORE_PREDICTION_SCORE_KEY] = getSafeNumberValueWithDefault(it) }
```

### 4. Event generator maps to HORIZONTAL_ELEMENT_SCORE
**`ContainerEventsGenerator.kt:1632-1634`**
```kotlin
storeLogging[STORE_PREDICTION_SCORE_KEY]?.let {
    horizontalElementScore = it.numberValue
    facetScore = it.numberValue
}
```
**`ContainerEventsGenerator.kt:2037-2041`** (cross-vertical CX homepage):
```kotlin
STORE_PREDICTION_SCORE(STORE_PREDICTION_SCORE_KEY, { b, v ->
    is Events.CrossVerticalHomePageFeedEvent.Builder -> b.setHorizontalElementScore(v.numberValue)
})
```

## What IS logged today
| Field | Value | Source |
|-------|-------|--------|
| `HORIZONTAL_ELEMENT_SCORE` | `finalScore` | `storePredictionScoresMap[id].finalScore` |
| `genai_embedding_similarity_score` | `embeddingScore` | PR #62113 (if merged) |
| `reranker_alpha` / `reranker_beta` | alpha/beta coeffs | `trackingPayload` via PR #62113 |

## What is NOT logged
| Missing Field | Value | Why it matters |
|---------------|-------|----------------|
| `combinedScore` | `finalScore^alpha * similarity^beta` | Actual signal that determines store order in carousel |

## Key Files
- `libraries/discovery/.../programmatic/GeneratedRecommendationCarouselService.kt`
- `libraries/discovery/.../programmatic/util/HomepageProgrammaticProductServiceUtil.kt`
- `libraries/domain-util/.../facets/adapters/StoreCarouselDataAdapter.kt`
- `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt`
- `libraries/domain-util/.../facets/LoggingConstants.kt` (STORE_PREDICTION_SCORE_KEY = "store_prediction_score", line 75)

## Snowflake Column Read Cost: Array vs Map

The existing `SCORE_MODIFIERS` column is a VARIANT containing a JSON **array** of `{name, value}` objects (from proto `repeated ScoreModifier`). Querying a specific modifier requires `LATERAL FLATTEN` — this explodes each row into N rows (one per array element), then filters. On a massive table like `cx_cross_vertical_homepage_feed`, this is expensive and has been flagged by stakeholders as a concern.

A proto `map<string, double>` serializes as a JSON **object** and lands in Snowflake as a VARIANT. Keys are directly addressable:
```sql
-- Array (FLATTEN required, expensive):
SELECT f.value:value::DOUBLE FROM table, LATERAL FLATTEN(input => SCORE_MODIFIERS) f WHERE f.value:name = 'key'

-- Map (direct access, same cost as scalar column):
SELECT STORE_RANKING_METRICS:key_name::DOUBLE FROM table
```

This is why `store_ranking_metrics` uses `map<string, double>` — direct key access, no flatten, no row multiplication. The existing `score_modifiers` array stays untouched (consumed by prod ML jobs).

## Solution

See [design.md](design.md) for the RankingSignalCollector architecture that solves this gap universally across all carousel types.

## Related
- `brain/genai-reranker-logging/` — PR #62113 adds embeddingScore + alpha/beta logging
- With alpha, beta, finalScore, and embeddingScore all logged, combinedScore can be **recomputed** from Snowflake. But logging it directly would simplify analysis and avoid floating-point discrepancies.
