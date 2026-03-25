# Plan 1: Generic Score Modifier Pipeline Infrastructure

**Date**: 2026-03-25
**Depends on**: Nothing
**Blocks**: Plan 2 (genai_combined_score)

## Goal

Add a generic `scoreModifiers: Map<String, Double>` pipeline through `StoreEntity` → `StoreCarouselDataAdapter` → `ContainerEventsGenerator` so that any per-store score modifier can be logged to Snowflake's `SCORE_MODIFIERS` column by adding a single line at the source. No sandbox testing — unit tests only.

## Design Decisions

### `score_modifiers` vs `carousel_details`
- **`score_modifiers`** (proto `repeated ScoreModifier`): per-store values that vary across stores within a carousel (e.g. `sr_multiplier`, `genai_combined_score`)
- **`carousel_details`** (trackingPayload): per-carousel constants, same for every store (e.g. `alpha`, `beta`, `carousel_id`)

### Fix forward, no migration
Existing typed fields (`genaiEmbeddingSimilarityScore`, `srMultiplier`, etc.) stay as-is. The generic pipeline runs alongside them. New scores use the map going forward.

## Files to Modify (3 production + tests)

| # | File | Change |
|---|------|--------|
| 1 | `StoreEntity.kt` | Add `scoreModifiers: Map<String, Double> = emptyMap()` + Builder + docs |
| 2 | `StoreCarouselDataAdapter.kt` | Add generic loop to write all `scoreModifiers` entries to logging map + docs |
| 3 | `ContainerEventsGenerator.kt` | After `LoggedValue.assign()`, iterate unhandled numeric entries and emit `ScoreModifier` for each + docs |

## Code Changes

### 1. StoreEntity — add generic scoreModifiers map

**File:** `libraries/platform/.../models/StoreEntity.kt`

Add after `genaiEmbeddingSimilarityScore` (line 119):
```kotlin
/**
 * Generic map of per-store score modifiers for Iguazu logging.
 * Values are automatically written to the store logging map and emitted as
 * ScoreModifier entries in cx_cross_vertical_homepage_feed events.
 *
 * Use this for NEW per-store numeric signals instead of adding typed fields.
 * Key = modifier name (appears as ScoreModifier.name in Snowflake SCORE_MODIFIERS column).
 * Value = numeric score.
 */
val scoreModifiers: Map<String, Double> = emptyMap(),
```
Plus corresponding Builder field and `build()` passthrough.

### 2. StoreCarouselDataAdapter — generic loop

**File:** `libraries/domain-util/.../facets/adapters/StoreCarouselDataAdapter.kt`

After existing explicit score blocks (line ~2521), add:
```kotlin
// Generic score modifiers — automatically logged without per-field wiring.
// To add a new score modifier, populate StoreEntity.scoreModifiers at the source.
store.scoreModifiers.forEach { (key, value) ->
    map[key] = getSafeNumberValueWithDefault(value)
}
```

### 3. ContainerEventsGenerator — generic ScoreModifier emission

**File:** `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt`

After each `LoggedValue.assign(storeLogging, this, STORE_LOGGED_VALUES)` call (lines 1631, 1653, 1734, and the programmatic subclass), add:
```kotlin
// Emit dynamic score modifiers not already handled by LoggedValue.
// To log a new per-store score, add it to StoreEntity.scoreModifiers — no changes needed here.
val handledKeys = currentLoggedValues.map { it.key }.toSet()
storeLogging.forEach { (key, value) ->
    if (key !in handledKeys && value.hasNumberValue()) {
        when (this) {
            is Events.CrossVerticalHomePageFeedEvent.Builder ->
                addScoreModifiers(
                    Events.CrossVerticalHomePageFeedEvent.ScoreModifier.newBuilder()
                        .setName(key).setValue(value.numberValue).build(),
                )
        }
    }
}
```

Where `currentLoggedValues` is the `STORE_LOGGED_VALUES` list used in that specific `assign()` call (base or subclass).

## Unit Tests

### StoreCarouselDataAdapterTest
- `StoreEntity` with `scoreModifiers = mapOf("test_score" to 0.75)` → logging map contains `"test_score"` → `0.75`
- `StoreEntity` with empty `scoreModifiers` → no extra entries in logging map

### ContainerEventsGeneratorTest
- Store logging map with an entry not in `STORE_LOGGED_VALUES` and `hasNumberValue() = true` → emitted as `ScoreModifier` in CX event
- Store logging map with only `STORE_LOGGED_VALUES` keys → no extra `ScoreModifier` entries
- Store logging map with non-numeric value not in `STORE_LOGGED_VALUES` → skipped (not emitted as `ScoreModifier`)

## Run Tests
```bash
./gradlew :libraries:domain-util:test --tests "*StoreCarouselDataAdapterTest*"
./gradlew :libraries:domain-util:test --tests "*ContainerEventsGeneratorTest*"
```

## PR
- Title: `Add generic scoreModifiers pipeline for per-store Iguazu logging`
- Description: Adds `StoreEntity.scoreModifiers` map with automatic forwarding through adapter and event generator. New per-store scores only need to populate this map at the source — no adapter/generator/constants changes required.
