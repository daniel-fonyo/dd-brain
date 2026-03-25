# Plan 1: Store Ranking Metrics Infrastructure

**Date**: 2026-03-25
**Depends on**: Nothing
**Blocks**: Plan 2 (genai_combined_score)

## Goal

Add a generic per-store ranking metrics pipeline: `StoreEntity` → `StoreCarouselDataAdapter` → `ContainerEventsGenerator` → new `store_ranking_metrics` Snowflake column. Adding a new per-store ranking signal becomes a single-line change at the source. No sandbox testing — unit tests only.

## Design Decisions

### New `store_ranking_metrics` column (not reusing `score_modifiers`)
The existing `score_modifiers` column is a JSON **array** (`repeated ScoreModifier` proto) consumed by production ML jobs. It requires expensive `LATERAL FLATTEN` to query individual values.

The new `store_ranking_metrics` column uses a proto `map<string, double>` which serializes as a JSON **object**. Snowflake can access keys directly (`column:key_name::DOUBLE`) — no flatten, no row explosion. Same cost as reading a scalar column.

The two columns are completely independent. Existing `score_modifiers` is untouched. New per-store signals go only into `store_ranking_metrics`.

### Per-store signals vs carousel-level constants
- **`store_ranking_metrics`** (new): per-store values that vary across stores within a carousel (e.g. `genai_combined_score`, `genai_embedding_similarity_score`)
- **`carousel_details`** (existing trackingPayload): per-carousel constants, same for every store (e.g. `alpha`, `beta`, `carousel_id`)

### Fix forward, no migration
Existing typed fields (`genaiEmbeddingSimilarityScore`, `srMultiplier`, etc.) and existing `score_modifiers` stay as-is. The new pipeline runs alongside them. New signals use the map going forward.

## Files to Modify (4 production + tests)

| # | File | Repo | Change |
|---|------|------|--------|
| 1 | `events.proto` | services-protobuf | Add `map<string, double> store_ranking_metrics = 86` to `CrossVerticalHomePageFeedEvent` |
| 2 | `StoreEntity.kt` | feed-service | Add `scoreModifiers: Map<String, Double> = emptyMap()` + Builder + docs |
| 3 | `StoreCarouselDataAdapter.kt` | feed-service | Add generic loop to write all `scoreModifiers` entries to logging map + docs |
| 4 | `ContainerEventsGenerator.kt` | feed-service | After `LoggedValue.assign()`, iterate unhandled numeric entries and populate `store_ranking_metrics` map on proto + docs |

## Code Changes

### 1. events.proto — add map field

**File:** `services-protobuf/protos/feed_service/events.proto`

In `CrossVerticalHomePageFeedEvent`, after `video_id` (field 84):
```protobuf
// Per-store ranking metrics as a flat key-value map.
// Queryable in Snowflake via STORE_RANKING_METRICS:key_name::DOUBLE (no FLATTEN needed).
// New per-store signals should be added here via StoreEntity.scoreModifiers in feed-service.
// Existing score_modifiers (field 59) is unchanged and consumed by production ML jobs.
map<string, double> store_ranking_metrics = 86;
//next id: 87
```

### 2. StoreEntity — add generic scoreModifiers map

**File:** `libraries/platform/.../models/StoreEntity.kt`

Add after `genaiEmbeddingSimilarityScore` (line 119):
```kotlin
/**
 * Generic map of per-store ranking metrics for Iguazu logging.
 * Entries are automatically forwarded through the logging pipeline and emitted
 * into the `store_ranking_metrics` map field on cx_cross_vertical_homepage_feed events.
 *
 * Queryable in Snowflake as: STORE_RANKING_METRICS:key_name::DOUBLE (no FLATTEN).
 *
 * USE THIS for new per-store numeric signals instead of adding typed fields to StoreEntity
 * and hardcoded LoggedValue enum entries. Just populate this map at the source.
 */
val scoreModifiers: Map<String, Double> = emptyMap(),
```
Plus corresponding Builder field and `build()` passthrough.

### 3. StoreCarouselDataAdapter — generic loop

**File:** `libraries/domain-util/.../facets/adapters/StoreCarouselDataAdapter.kt`

After existing explicit score blocks (line ~2521), add:
```kotlin
// Generic per-store ranking metrics — automatically logged without per-field wiring.
// To add a new metric, populate StoreEntity.scoreModifiers at the source.
// These flow to the store_ranking_metrics Snowflake column (not score_modifiers).
store.scoreModifiers.forEach { (key, value) ->
    map[key] = getSafeNumberValueWithDefault(value)
}
```

### 4. ContainerEventsGenerator — populate store_ranking_metrics map

**File:** `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt`

After each `LoggedValue.assign(storeLogging, this, STORE_LOGGED_VALUES)` call, add:
```kotlin
// Populate store_ranking_metrics map with dynamic per-store signals.
// These come from StoreEntity.scoreModifiers via StoreCarouselDataAdapter.
// To add a new metric, populate StoreEntity.scoreModifiers at the source — no changes needed here.
val handledKeys = currentLoggedValues.map { it.key }.toSet()
storeLogging.forEach { (key, value) ->
    if (key !in handledKeys && value.hasNumberValue()) {
        when (this) {
            is Events.CrossVerticalHomePageFeedEvent.Builder ->
                putStoreRankingMetrics(key, value.numberValue)
        }
    }
}
```

Where `currentLoggedValues` is the `STORE_LOGGED_VALUES` list used in that specific `assign()` call (base or subclass).

Note: `putStoreRankingMetrics` is the generated method for proto `map<string, double>` fields.

## Unit Tests

### StoreCarouselDataAdapterTest
- `StoreEntity` with `scoreModifiers = mapOf("test_score" to 0.75)` → logging map contains `"test_score"` → `0.75`
- `StoreEntity` with empty `scoreModifiers` → no extra entries in logging map

### ContainerEventsGeneratorTest
- Store logging map with an entry not in `STORE_LOGGED_VALUES` and `hasNumberValue() = true` → present in `storeRankingMetricsMap` on CX event builder
- Store logging map with only `STORE_LOGGED_VALUES` keys → `storeRankingMetricsMap` is empty
- Store logging map with non-numeric value not in `STORE_LOGGED_VALUES` → not in `storeRankingMetricsMap`

## Run Tests
```bash
./gradlew :libraries:domain-util:test --tests "*StoreCarouselDataAdapterTest*"
./gradlew :libraries:domain-util:test --tests "*ContainerEventsGeneratorTest*"
```

## PR
- Title: `Add store_ranking_metrics pipeline for per-store Iguazu logging`
- Description: Adds `StoreEntity.scoreModifiers` map with automatic forwarding to a new `store_ranking_metrics` proto map field. Queryable in Snowflake as `STORE_RANKING_METRICS:key::DOUBLE` without FLATTEN. Existing `score_modifiers` is untouched.
