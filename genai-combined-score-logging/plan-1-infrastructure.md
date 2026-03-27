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

## Files to Modify (5 production + tests)

| # | File | Repo | Change |
|---|------|------|--------|
| 3 | `events.proto` | services-protobuf | Add `map<string, string> ranking_signals = 86` to `CrossVerticalHomePageFeedEvent` |
| 4 | `BaseDiscoveryContext.kt` | feed-service | Add `val rankingSignalCollector` with default getter to `BaseDiscoveryProductContext` interface |
| 5 | `ExploreContext.kt` | feed-service | Override with stored `RankingSignalCollector` instance |
| 6 | `StoreCarouselDataAdapter.kt` | feed-service | One-line call to `RankingSignalWriter.writeEntitySignals()` + `writeCarouselSignals()` |
| 7 | `ContainerEventsGenerator.kt` | feed-service | Generic loop: unhandled logging map keys → `ranking_signals` proto map |

## Code Changes

### 1. RankingSignalCollector — new file (interface + impl + no-op)

**Location**: `libraries/platform/src/main/kotlin/com/doordash/consumer/feed/platform/ranking/RankingSignalCollector.kt`

```kotlin
package com.doordash.consumer.feed.platform.ranking

import java.util.concurrent.ConcurrentHashMap

/** Interface for collecting ranking signals during the homepage pipeline. */
interface RankingSignalCollector {
    fun record(carouselId: String, entityId: Long, key: String, value: Any)
    fun record(carouselId: String, key: String, value: Any)
    fun forEntity(carouselId: String, entityId: Long): Map<String, String>
    fun forCarousel(carouselId: String): Map<String, String>

    companion object {
        /** No-op singleton. Zero allocation, zero side effects. Default for non-homepage contexts. */
        val NOOP: RankingSignalCollector = object : RankingSignalCollector {
            override fun record(carouselId: String, entityId: Long, key: String, value: Any) {}
            override fun record(carouselId: String, key: String, value: Any) {}
            override fun forEntity(carouselId: String, entityId: Long) = emptyMap<String, String>()
            override fun forCarousel(carouselId: String) = emptyMap<String, String>()
        }
    }
}

/** Real implementation. Thread-safe via ConcurrentHashMap. One per ExploreContext (one per request). */
class DefaultRankingSignalCollector : RankingSignalCollector {
    private val entitySignals = ConcurrentHashMap<String, ConcurrentHashMap<Long, ConcurrentHashMap<String, String>>>()
    private val carouselSignals = ConcurrentHashMap<String, ConcurrentHashMap<String, String>>()

    override fun record(carouselId: String, entityId: Long, key: String, value: Any) {
        entitySignals
            .getOrPut(carouselId) { ConcurrentHashMap() }
            .getOrPut(entityId) { ConcurrentHashMap() }[key] = value.toString()
    }

    override fun record(carouselId: String, key: String, value: Any) {
        carouselSignals
            .getOrPut(carouselId) { ConcurrentHashMap() }[key] = value.toString()
    }

    override fun forEntity(carouselId: String, entityId: Long): Map<String, String> =
        entitySignals[carouselId]?.get(entityId) ?: emptyMap()

    override fun forCarousel(carouselId: String): Map<String, String> =
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
 * Keys are prefixed with [RANKING_SIGNAL_PREFIX] ("rs:") so ContainerEventsGenerator
 * can distinguish ranking signals from existing logging keys and forward only
 * rs:-prefixed entries to the ranking_signals proto map.
 *
 * The prefix is stripped before writing to the proto — Snowflake sees clean key names.
 */
object RankingSignalWriter {

    const val RANKING_SIGNAL_PREFIX = "rs:"

    fun writeEntitySignals(
        collector: RankingSignalCollector,
        carouselId: String,
        entityId: Long,
        map: MutableMap<String, Value>,
    ) {
        collector.forEntity(carouselId, entityId).forEach { (key, value) ->
            map[RANKING_SIGNAL_PREFIX + key] = toValue(value)
        }
    }

    fun writeCarouselSignals(
        collector: RankingSignalCollector,
        carouselId: String,
        map: MutableMap<String, Value>,
    ) {
        collector.forCarousel(carouselId).forEach { (key, value) ->
            map[RANKING_SIGNAL_PREFIX + key] = toValue(value)
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

### 4. BaseDiscoveryProductContext — add interface default (no-op)

**File**: `libraries/sdk-dex/src/main/kotlin/com/doordash/consumer/discovery/sdk/dex/models/context/BaseDiscoveryContext.kt`

In the `BaseDiscoveryProductContext` interface, add:
```kotlin
val rankingSignalCollector: RankingSignalCollector
    get() = RankingSignalCollector.NOOP  // safe no-op singleton, zero allocation
```

### 5. ExploreContext — override with real instance

**File**: `libraries/sdk-dex/src/main/kotlin/com/doordash/consumer/discovery/sdk/dex/models/context/ExploreContext.kt`

Add to the data class constructor:
```kotlin
override val rankingSignalCollector: RankingSignalCollector = DefaultRankingSignalCollector(),
```

This ensures the same mutable collector instance is used for the entire request. Survives `.copy()` (shallow copy preserves the reference). Accessible everywhere via `context.rankingSignalCollector` — whether `context` is typed as `ExploreContext` or `BaseDiscoveryProductContext`.

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

### 7. ContainerEventsGenerator — prefix-based forwarding

**File**: `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt`

After `LoggedValue.assign(storeLogging, this, STORE_LOGGED_VALUES)`, add prefix-based forwarding:

```kotlin
// Forward ranking signals (rs:-prefixed) to ranking_signals proto map.
// Only rs:-prefixed keys are forwarded — existing unhandled keys are not affected.
// Prefix is stripped so Snowflake sees clean key names.
storeLogging.forEach { (key, value) ->
    if (key.startsWith(RankingSignalWriter.RANKING_SIGNAL_PREFIX)) {
        val signalKey = key.removePrefix(RankingSignalWriter.RANKING_SIGNAL_PREFIX)
        when (this) {
            is Events.CrossVerticalHomePageFeedEvent.Builder ->
                putRankingSignals(
                    signalKey,
                    if (value.hasNumberValue()) value.numberValue.toString() else value.stringValue,
                )
        }
    }
}
```

**Why prefix instead of forwarding all unhandled keys?** The adapter's logging map contains ~40+ keys. `LoggedValue.assign()` handles ~66 known keys. Many existing keys are intentionally unhandled (like `PAGE_KEY`, `BADGES_LOGGING_KEY`). Forwarding ALL unhandled keys would pollute `ranking_signals` with non-ranking data. The `rs:` prefix cleanly separates ranking signals from everything else.

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

1. ~~Confirm `ExploreContext` is the right host for the collector~~ — **Confirmed**. ExploreContext is threaded through every pipeline path. Collector goes on `BaseDiscoveryProductContext` interface (with default) so it's accessible even in methods that take the parent interface.
2. Confirm the proto field number 86 is available (check latest `events.proto`).
3. Verify Iguazu auto-creates `RANKING_SIGNALS` VARIANT column from proto map field — or identify manual steps needed.
4. Confirm `currentLoggedValues` variable name in ContainerEventsGenerator (it may be called `STORE_LOGGED_VALUES` directly or passed as a parameter).

## PR
- **Title**: `Add RankingSignalCollector for universal ranking signal logging`
- **Description**: Adds a request-scoped collector + writer utility for logging per-entity and per-carousel ranking signals to Snowflake via a new `ranking_signals` proto map field. Any pipeline step records signals with `collector.record()`. No typed fields, no per-signal adapter wiring. First consumer: genai_combined_score (Plan 2).
