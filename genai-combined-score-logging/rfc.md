# RFC: Ranking Signal Logging Pipeline

**Author**: Daniel Fonyo
**Date**: 2026-03-27
**Status**: Draft

## Problem

When a store appears in position 3 of a GenAI carousel, we cannot answer _why_. The `combinedScore` that determined its rank was computed, used to sort, and thrown away. The only score that survives to Snowflake is the original ML prediction, not the signal that actually decided the order.

This is not unique to GenAI. Across the homepage pipeline, ranking steps compute scores, apply weights, and choose strategies. Most of this work is invisible by the time data reaches Iguazu. If you want to log a new signal today, you need to:

1. Add a typed field to a data class
2. Manually copy it through 2 to 3 carrier objects
3. Add a hardcoded adapter mapping
4. Add a `LoggedValue` enum entry in `ContainerEventsGenerator`

This is the pattern that grew `StoreEntity` to 226 fields. Each new signal costs a multi file PR and touches shared infrastructure. We want the cost to be **one line**.

## Solution

Three components. A collector accumulates signals. A writer flushes them. Two proto map fields carry them to Snowflake.

### How it works

**1. Record.** Any pipeline step, any carousel type:
```kotlin
context.rankingSignalCollector.record(collection.id, store.storeId(), "genai_combined_score", combinedScore)
context.rankingSignalCollector.record(collection.id, "reranker_alpha", alpha)
```

**2. Flush.** Each adapter, one line:
```kotlin
RankingSignalWriter.writeEntitySignals(context.rankingSignalCollector, carouselId, store.id, map)
RankingSignalWriter.writeCarouselSignals(context.rankingSignalCollector, carouselId, map)
```

**3. Emit.** `ContainerEventsGenerator` forwards prefixed keys to proto maps. No per signal wiring.

**4. Query.** Snowflake, direct key access:
```sql
SELECT
    RANKING_SIGNALS:genai_combined_score::DOUBLE,
    CAROUSEL_RANKING_SIGNALS:reranker_alpha::DOUBLE
FROM CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
WHERE IGUAZU_PARTITION_DATE = CURRENT_DATE()
```

### Components

**`RankingSignalCollector`** (interface). Request scoped, lives on `ExploreContext`. Two level storage: per entity signals (scores that vary per store) and per carousel signals (constants like alpha/beta). Thread safe via `ConcurrentHashMap`. `BaseDiscoveryProductContext` exposes it with a no op default so non homepage contexts pay zero cost.

**`RankingSignalWriter`** (static utility). Flushes signals from the collector into an adapter's logging map. Prefixes entity keys with `rs:` and carousel keys with `crs:` to namespace them from existing logging keys. Handles type detection (numbers vs strings).

**Proto fields** on `CrossVerticalHomePageFeedEvent`:
```protobuf
map<string, string> ranking_signals = 86;           // per entity
map<string, string> carousel_ranking_signals = 87;  // per carousel
```

These land as VARIANT columns in Snowflake with direct key access (`COLUMN:key::TYPE`). No `LATERAL FLATTEN` like the existing `SCORE_MODIFIERS` array.

**`ContainerEventsGenerator` forwarding.** After `LoggedValue.assign()`, a loop forwards `rs:` prefixed keys to `ranking_signals` and `crs:` prefixed keys to `carousel_ranking_signals`, stripping the prefix. Existing logging keys are unaffected.

### Data flow

```
Pipeline step                  Adapter                    Iguazu
    |                              |                         |
    |  collector.record()          |                         |
    |  (any phase, any type)       |                         |
    v                              v                         v
+------------+  forEntity()  +----------+  rs:/crs:   +----------+
| Collector  |-------------->|  Writer  |------------>| Events   |--> Snowflake
| on Context |  forCarousel() |  (flush) |  prefix     | Generator|
+------------+               +----------+  routing    +----------+
```

Signals recorded at any phase survive to Snowflake because the collector is a mutable object on the request context, the same instance from start to finish.

### Why this shape

**Collector on `BaseDiscoveryProductContext`**, not on collection types. The homepage has 6+ collection types (`LiteStoreCollection`, `DealCarousel`, `ItemCarousel`, etc.) with no shared base. `ExploreContext` is the one object threaded through every path. The interface default is a no op singleton, zero cost for non homepage callers.

**Writer as a utility**, not adapter polymorphism. The four adapters (`StoreCarouselDataAdapter`, `DealCarouselDataAdapter`, `ItemCarouselPageDataAdapter`, `AnnouncementsAdapter`) have no shared interface, different input types, and different output types. But they all build a `MutableMap<String, Value>` logging map. The writer hooks into that shared step.

**Prefix routing** (`rs:` / `crs:`), not "forward all unhandled keys". The adapter's logging map has around 40 keys that `ContainerEventsGenerator` intentionally drops. Forwarding all unhandled keys would pollute the new columns with badges, display URLs, and other non ranking data. The prefix gives ranking signals their own clean namespace.

**`map<string, string>`**, not `map<string, double>`. We log both numbers (scores, weights) and strings (model names, strategies). Snowflake casts transparently. `::DOUBLE` on a numeric string works.

## What This Change Does NOT Do

- **Does not migrate existing signals.** `SCORE_MODIFIERS`, `HORIZONTAL_ELEMENT_SCORE`, and other existing columns stay as is. This is additive.
- **Does not change StoreEntity or any entity class.** Signals bypass entities entirely, from collector to adapter to proto.
- **Does not wire all adapters.** Initial scope is `StoreCarouselDataAdapter` only. Deal, item, and announcement adapters get a one line addition when needed.
- **Does not replace `trackingPayload` or `CAROUSEL_DETAILS`.** Those continue to work for their current consumers.

## First Consumer: GenAI Combined Score

After the infrastructure ships, logging `combinedScore` is one file change:

```kotlin
// In GeneratedRecommendationCarouselService.rerankStoreByBlockReranking()
reRankedStoreScoreMap.forEach { (store, score) ->
    context.rankingSignalCollector.record(collection.id, store.storeId(), "genai_combined_score", score)
}
context.rankingSignalCollector.record(collection.id, "genai_reranker_alpha", alpha)
context.rankingSignalCollector.record(collection.id, "genai_reranker_beta", beta)
context.rankingSignalCollector.record(collection.id, "genai_ranking_model", "block_reranking")
```

No adapter changes. No event generator changes. No new typed fields.

## Files Changed

**New** (2):
- `RankingSignalCollector.kt`, interface + `DefaultRankingSignalCollector` + `NOOP` singleton
- `RankingSignalWriter.kt`, static utility with prefix logic

**Modified** (6):
- `BaseDiscoveryContext.kt`, add `rankingSignalCollector` with no op default to interface
- `ExploreContext.kt`, override with `DefaultRankingSignalCollector()`
- `events.proto`, add two `map<string, string>` fields
- `StoreCarouselDataAdapter.kt`, one line writer call in `generateStoreLogging()`
- `ContainerEventsGenerator.kt`, prefix based forwarding loop after `LoggedValue.assign()`
- `GeneratedRecommendationCarouselService.kt`, `collector.record()` calls (first consumer)

## Open

- **Iguazu column auto creation**: Verify that the new proto map fields automatically create `RANKING_SIGNALS` and `CAROUSEL_RANKING_SIGNALS` VARIANT columns in Snowflake, or identify manual steps needed.
