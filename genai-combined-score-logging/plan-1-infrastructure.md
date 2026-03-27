# Plan 1: RankingSignalCollector Infrastructure

**Date**: 2026-03-27 (revised)
**Depends on**: Nothing
**Blocks**: Plan 2 (genai_combined_score)
**Design doc**: [design.md](design.md)

## Goal

Build the universal ranking signal logging pipeline: `RankingSignalCollector` (on ExploreContext) + `RankingSignalWriter` (utility) + proto `map<string, string> ranking_signals` field + `ContainerEventsGenerator` generic forwarding. After this, any pipeline step can log a ranking signal to Snowflake with a single `collector.record()` call.

## Files to Create (2 new)

| # | File | Repo | Description |
|---|------|------|-------------|
| 1 | `RankingSignalCollector.kt` | feed-service | Mutable request-scoped collector |
| 2 | `RankingSignalWriter.kt` | feed-service | Static utility — flushes signals into adapter logging maps |

## Files to Modify (4 production + tests)

| # | File | Repo | Change |
|---|------|------|--------|
| 3 | `events.proto` | services-protobuf | Add `map<string, string> ranking_signals = 86` to `CrossVerticalHomePageFeedEvent` |
| 4 | `ExploreContext.kt` (or equivalent) | feed-service | Add `val rankingSignalCollector: RankingSignalCollector = RankingSignalCollector()` |
| 5 | `StoreCarouselDataAdapter.kt` | feed-service | One-line call to `RankingSignalWriter.writeEntitySignals()` + `writeCarouselSignals()` |
| 6 | `ContainerEventsGenerator.kt` | feed-service | Generic loop: unhandled logging map keys → `ranking_signals` proto map |

## Code Changes

### 1. RankingSignalCollector — new file

**Location**: `libraries/platform/src/main/kotlin/com/doordash/consumer/feed/platform/ranking/RankingSignalCollector.kt`

```kotlin
package com.doordash.consumer.feed.platform.ranking

import java.util.concurrent.ConcurrentHashMap

/**
 * Request-scoped collector for ranking signals produced during the homepage pipeline.
 *
 * Any pipeline step (reranker, scorer, ranker) records signals via [record].
 * Adapters consume them via [forEntity] and [forCarousel] through [RankingSignalWriter].
 *
 * Signals flow to the `ranking_signals` proto map field on Iguazu events,
 * queryable in Snowflake as RANKING_SIGNALS:key_name::DOUBLE or ::VARCHAR.
 *
 * Thread-safe via ConcurrentHashMap (carousels are processed in parallel).
 */
class RankingSignalCollector {
    private val entitySignals = ConcurrentHashMap<String, ConcurrentHashMap<Long, ConcurrentHashMap<String, String>>>()
    private val carouselSignals = ConcurrentHashMap<String, ConcurrentHashMap<String, String>>()

    /** Record a per-entity signal (e.g., a store's combined score within a carousel). */
    fun record(carouselId: String, entityId: Long, key: String, value: Any) {
        entitySignals
            .getOrPut(carouselId) { ConcurrentHashMap() }
            .getOrPut(entityId) { ConcurrentHashMap() }[key] = value.toString()
    }

    /** Record a per-carousel signal (e.g., reranker alpha coefficient). */
    fun record(carouselId: String, key: String, value: Any) {
        carouselSignals
            .getOrPut(carouselId) { ConcurrentHashMap() }[key] = value.toString()
    }

    /** Get all signals for a specific entity in a carousel. */
    fun forEntity(carouselId: String, entityId: Long): Map<String, String> =
        entitySignals[carouselId]?.get(entityId) ?: emptyMap()

    /** Get all carousel-level signals. */
    fun forCarousel(carouselId: String): Map<String, String> =
        carouselSignals[carouselId] ?: emptyMap()
}
```

### 2. RankingSignalWriter — new file

**Location**: `libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/utils/logging/RankingSignalWriter.kt`

```kotlin
package com.doordash.consumer.feed.domainUtil.utils.logging

import com.doordash.consumer.feed.domainUtil.facets.adapters.AdapterUtil
import com.doordash.consumer.feed.platform.ranking.RankingSignalCollector
import com.google.protobuf.Value

/**
 * Flushes ranking signals from [RankingSignalCollector] into an adapter's logging map.
 *
 * Each adapter calls this with one line. No per-signal wiring needed.
 * Numeric-parseable values become NumberValue; others become StringValue.
 */
object RankingSignalWriter {

    fun writeEntitySignals(
        collector: RankingSignalCollector,
        carouselId: String,
        entityId: Long,
        map: MutableMap<String, Value>,
    ) {
        collector.forEntity(carouselId, entityId).forEach { (key, value) ->
            map[key] = toValue(value)
        }
    }

    fun writeCarouselSignals(
        collector: RankingSignalCollector,
        carouselId: String,
        map: MutableMap<String, Value>,
    ) {
        collector.forCarousel(carouselId).forEach { (key, value) ->
            map[key] = toValue(value)
        }
    }

    private fun toValue(value: String): Value {
        val doubleVal = value.toDoubleOrNull()
        return if (doubleVal != null) {
            AdapterUtil.getSafeNumberValueWithDefault(doubleVal)
        } else {
            AdapterUtil.getStringValue(value)
        }
    }
}
```

### 3. events.proto — add map field

**File**: `services-protobuf/protos/feed_service/events.proto`

In `CrossVerticalHomePageFeedEvent`, after `video_id` (field 84):
```protobuf
// Per-entity ranking signals as a flat key-value map.
// Populated by RankingSignalCollector in feed-service.
// Queryable in Snowflake as RANKING_SIGNALS:key_name::DOUBLE or ::VARCHAR (no FLATTEN).
// Existing score_modifiers (field 59) is unchanged and consumed by production ML jobs.
map<string, string> ranking_signals = 86;
//next id: 87
```

### 4. ExploreContext — add collector field

**File**: Exact location TBD — need to confirm which class is the request-scoped context threaded through the full pipeline. Likely `ExploreContext` in `libraries/platform/`.

```kotlin
val rankingSignalCollector: RankingSignalCollector = RankingSignalCollector(),
```

### 5. StoreCarouselDataAdapter — one-line writer call

**File**: `libraries/domain-util/.../facets/adapters/StoreCarouselDataAdapter.kt`

In `StoreCarouselDataAdapterUtil.generateStoreRowLogging()` (or equivalent method that builds the per-store logging map), after existing score mappings:

```kotlin
// Flush ranking signals from collector into logging map.
// New signals are added via context.rankingSignalCollector.record() — no changes needed here.
RankingSignalWriter.writeEntitySignals(
    context.rankingSignalCollector, storeCarousel.id, store.id, map
)
RankingSignalWriter.writeCarouselSignals(
    context.rankingSignalCollector, storeCarousel.id, map
)
```

### 6. ContainerEventsGenerator — generic forwarding

**File**: `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt`

After `LoggedValue.assign(storeLogging, this, STORE_LOGGED_VALUES)`, add generic pass:

```kotlin
// Forward unhandled logging map entries to ranking_signals proto map.
// These come from RankingSignalCollector via RankingSignalWriter.
val handledKeys = currentLoggedValues.map { it.key }.toSet()
storeLogging.forEach { (key, value) ->
    if (key !in handledKeys) {
        when (this) {
            is Events.CrossVerticalHomePageFeedEvent.Builder ->
                putRankingSignals(
                    key,
                    if (value.hasNumberValue()) value.numberValue.toString() else value.stringValue,
                )
        }
    }
}
```

## Unit Tests

### RankingSignalCollectorTest
- `record()` entity signal → `forEntity()` returns it
- `record()` carousel signal → `forCarousel()` returns it
- Multiple carousels / entities → signals are isolated
- Unrecorded carousel/entity → returns empty map

### RankingSignalWriterTest
- Numeric string `"0.75"` → written as `NumberValue(0.75)`
- Non-numeric string `"block_reranking"` → written as `StringValue("block_reranking")`
- Empty collector → no entries written
- Multiple signals → all written

### StoreCarouselDataAdapterTest
- Store with signals in collector → logging map contains those signals
- Store without signals → no extra entries in logging map

### ContainerEventsGeneratorTest
- Logging map entry not in `STORE_LOGGED_VALUES` → present in `rankingSignalsMap` on proto builder
- Only `STORE_LOGGED_VALUES` keys → `rankingSignalsMap` is empty
- Non-numeric value → stored as string in `rankingSignalsMap`

## Run Tests
```bash
./gradlew :libraries:platform:test --tests "*RankingSignalCollectorTest*"
./gradlew :libraries:domain-util:test --tests "*RankingSignalWriterTest*"
./gradlew :libraries:domain-util:test --tests "*StoreCarouselDataAdapterTest*"
./gradlew :libraries:domain-util:test --tests "*ContainerEventsGeneratorTest*"
```

## TODO: Verify Before Plan 2

1. Confirm `ExploreContext` is the right host for the collector (check it's accessible from all pipeline steps and all adapters).
2. Confirm the proto field number 86 is available (check latest `events.proto`).
3. Verify Iguazu auto-creates `RANKING_SIGNALS` VARIANT column from proto map field — or identify manual steps needed.
4. Confirm `currentLoggedValues` variable name in ContainerEventsGenerator (it may be called `STORE_LOGGED_VALUES` directly or passed as a parameter).

## PR
- **Title**: `Add RankingSignalCollector for universal ranking signal logging`
- **Description**: Adds a request-scoped collector + writer utility for logging per-entity and per-carousel ranking signals to Snowflake via a new `ranking_signals` proto map field. Any pipeline step records signals with `collector.record()`. No typed fields, no per-signal adapter wiring. First consumer: genai_combined_score (Plan 2).
