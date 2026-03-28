# RFC: Homepage Ranking Signal Logging Framework

**Author**: Daniel Fonyo
**Date**: 2026-03-27
**Status**: Draft
**Design doc**: [design.md](design.md)
**Gap analysis**: [analysis.md](analysis.md)

## Problem

Adding a new logging field to `cx_cross_vertical_homepage_feed` is painful. To log a single new value, an engineer has to:

1. Add a typed field to a data class (e.g. `GeneratedRecommendationStoreInfo` or `StoreEntity`)
2. Manually copy that field through 2 to 3 carrier objects during the decoration phase (`LiteStoreCollection` to `StoreEntity` via `HomepageProgrammaticProductServiceUtil.updateStoreEntity()`)
3. Add a hardcoded mapping in `StoreCarouselDataAdapter.generateStoreLogging()` to write the field into the logging map
4. Add a `LoggedValue` enum entry in `ContainerEventsGenerator` to map the logging key to a proto field on `CrossVerticalHomePageFeedEvent`
5. Write unit tests for each of these layers

That is a minimum of 4 production files changed across `libraries/discovery`, `libraries/platform`, `libraries/domain-util`, and `services-protobuf`. The PR touches shared infrastructure owned by different teams. The engineer has to trace the full path from where the value is computed to where it lands in Snowflake, stringing it through a chain of classes where it is easy to miss a step or wire it incorrectly. For someone not deeply familiar with the system, this can take two weeks.

This is the pattern that grew `StoreEntity` to 226 constructor parameters. It does not scale.

### Concrete Example

GenAI carousels rerank stores using a combined score: `finalScore^alpha * similarity^beta`. This score determines the final store order. After sorting, the score is discarded. The only value that reaches Snowflake is the original `finalScore` via `HORIZONTAL_ELEMENT_SCORE`. The actual reranking signal, the one that decided the order, is invisible. Logging it through the current pattern would require all 4 steps above.

This RFC introduces a framework where logging a new signal costs **one line**.

## Goals

1. **Any ranking step can log any signal with a single `record()` call.** No typed fields, no adapter wiring, no enum entries. The framework handles everything downstream.
2. **Works for all homepage carousel types.** Store carousels, deal carousels, item carousels, announcements. Not limited to a single carousel type or ranking step.
3. **Supports both entity level and carousel level signals.** Per store values (like a combined score that varies across stores) and per carousel values (like the alpha/beta coefficients used by the reranker).
4. **Supports numbers and strings.** Scores, weights, multipliers (numeric) and model names, strategy labels, experiment variants (string).
5. **Queryable in Snowflake without LATERAL FLATTEN.** Direct key access on a VARIANT column.

## Scope

The initial proto wiring targets `CrossVerticalHomePageFeedEvent`, which populates the `cx_cross_vertical_homepage_feed` table. However, the `RankingSignalCollector` and `RankingSignalWriter` are generic abstractions not tied to any specific table or event type. Extending to other event types is a matter of adding the same two map fields to their proto definitions and the same prefix forwarding loop in their event generators.

## Non Goals

- **No migration of existing logged fields.** `SCORE_MODIFIERS`, `HORIZONTAL_ELEMENT_SCORE`, and all other existing Iguazu columns are untouched. The new framework runs alongside them. Migration of existing signals into the new columns can be future work and would be straightforward since it only requires adding `record()` calls at each signal's source.
- **No changes to StoreEntity or any entity data class.** Signals flow directly from the collector to the adapter's logging map, bypassing entity objects entirely.
- **No changes to the existing `LoggedValue` enum or `STORE_LOGGED_VALUES` list.** The new framework operates independently.
- **No replacement of `trackingPayload` or `CAROUSEL_DETAILS`.** Those continue to serve their current consumers.

## Solution

### Overview

The framework has three components:

1. **`RankingSignalCollector`**: a request scoped object that accumulates ranking signals throughout the pipeline. Any step in the pipeline records signals by calling `collector.record()`. The collector lives on `ExploreContext`, which is threaded through every path of the homepage pipeline.

2. **`RankingSignalWriter`**: a static utility called by each adapter with one line. It reads from the collector and writes signals into the adapter's existing logging map, prefixed to distinguish them from other logging keys.

3. **Two new proto map fields** on `CrossVerticalHomePageFeedEvent`: `ranking_signals` (per entity) and `carousel_ranking_signals` (per carousel). These become two new VARIANT columns in Snowflake with direct key access.

### Recording Signals

Any pipeline step that has access to the request context can record signals. This works at any phase: collection creation, reranking, unified ranking, decoration, or serialization.

Per entity signals (values that vary per store within a carousel):
```kotlin
context.rankingSignalCollector.record(carousel.id, store.storeId(), "genai_combined_score", combinedScore)
context.rankingSignalCollector.record(carousel.id, store.storeId(), "sr_multiplier", srMultiplier)
```

Per carousel signals (values that are constant across all stores in a carousel):
```kotlin
context.rankingSignalCollector.record(carousel.id, "reranker_alpha", alpha)
context.rankingSignalCollector.record(carousel.id, "ranking_model", "block_reranking_v2")
```

The collector stores these in memory, keyed by carousel ID and entity ID. It is thread safe via `ConcurrentHashMap` since the homepage pipeline processes carousels in parallel.

### Flushing to the Logging Map

Each adapter already builds a `MutableMap<String, Value>` as part of its logging generation. The writer hooks into this existing step with one line per adapter:

```kotlin
// In StoreCarouselDataAdapter.generateStoreLogging(), before the existing context.loggingMap merge
RankingSignalWriter.writeEntitySignals(context.rankingSignalCollector, carouselId, store.id, map)
RankingSignalWriter.writeCarouselSignals(context.rankingSignalCollector, carouselId, map)
```

The writer prefixes entity level keys with `rs:` and carousel level keys with `crs:` before writing them into the logging map. This prefix serves as a namespace. The existing logging map contains around 40 keys (store name, delivery fee, badges, etc.) that are not ranking signals. The prefix ensures only ranking signals are forwarded to the new proto fields, and existing keys continue to behave exactly as they do today.

### Emitting to Proto

`ContainerEventsGenerator` already iterates the logging map to populate proto event fields via `LoggedValue.assign()`. After that existing step, a new loop forwards prefixed keys to the new proto map fields:

```kotlin
storeLogging.forEach { (key, value) ->
    val stringVal = if (value.hasNumberValue()) value.numberValue.toString() else value.stringValue
    when {
        key.startsWith("rs:") -> {
            when (this) {
                is Events.CrossVerticalHomePageFeedEvent.Builder ->
                    putRankingSignals(key.removePrefix("rs:"), stringVal)
            }
        }
        key.startsWith("crs:") -> {
            when (this) {
                is Events.CrossVerticalHomePageFeedEvent.Builder ->
                    putCarouselRankingSignals(key.removePrefix("crs:"), stringVal)
            }
        }
    }
}
```

The prefix is stripped before writing to the proto. Snowflake sees clean key names.

### Proto Schema

Two new fields on `CrossVerticalHomePageFeedEvent`:

```protobuf
// Per entity ranking signals. Values that vary per store within a carousel.
// Example keys: genai_combined_score, sr_multiplier, taste_affinity_score
// Queryable in Snowflake: RANKING_SIGNALS:key_name::DOUBLE or ::VARCHAR
map<string, string> ranking_signals = 86;

// Per carousel ranking signals. Values constant across all stores in a carousel.
// Example keys: reranker_alpha, reranker_beta, ranking_model, ranking_strategy
// Queryable in Snowflake: CAROUSEL_RANKING_SIGNALS:key_name::DOUBLE or ::VARCHAR
map<string, string> carousel_ranking_signals = 87;
```

Both use `map<string, string>` to support numeric and string values in a single column. Snowflake casts transparently: `RANKING_SIGNALS:genai_combined_score::DOUBLE` works on a string that parses as a number.

### Snowflake: For ML Engineers

The new columns land as VARIANT types with direct key access. No `LATERAL FLATTEN` required, unlike the existing `SCORE_MODIFIERS` array.

**Querying the new columns:**
```sql
SELECT
    RANKING_SIGNALS:genai_combined_score::DOUBLE AS combined_score,
    RANKING_SIGNALS:sr_multiplier::DOUBLE AS sr_mult,
    CAROUSEL_RANKING_SIGNALS:reranker_alpha::DOUBLE AS alpha,
    CAROUSEL_RANKING_SIGNALS:ranking_model::VARCHAR AS model,
    STORE_ID,
    FACET_NAME
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
WHERE IGUAZU_PARTITION_DATE = CURRENT_DATE()
  AND IGUAZU_PARTITION_HOUR >= HOUR(CURRENT_TIMESTAMP()) - 2
  AND RANKING_SIGNALS IS NOT NULL
```

The existing `SCORE_MODIFIERS` column stores signals as a JSON array, which requires `LATERAL FLATTEN` to query. That operation explodes every array element into its own row before filtering, making queries expensive. The new columns store signals as JSON objects with direct key access (`COLUMN:key::DOUBLE`), which avoids the row multiplication entirely and performs comparably to reading a regular scalar column. `SCORE_MODIFIERS` and its consumers are completely unaffected. The new columns exist independently.

### Where the Collector Lives

`RankingSignalCollector` is an interface with two implementations:

- **`DefaultRankingSignalCollector`**: the real implementation, thread safe, used by `ExploreContext` on the homepage path.
- **`RankingSignalCollector.NOOP`**: a singleton that does nothing. Zero allocation, zero overhead.

The interface is exposed on `BaseDiscoveryProductContext` with a default that returns `NOOP`:

```kotlin
interface BaseDiscoveryProductContext : RankerContext {
    val rankingSignalCollector: RankingSignalCollector
        get() = RankingSignalCollector.NOOP
}
```

`ExploreContext` overrides this with a real stored instance:

```kotlin
data class ExploreContext(
    override val rankingSignalCollector: RankingSignalCollector = DefaultRankingSignalCollector(),
    // ... existing fields ...
) : BaseDiscoveryProductContext
```

This means every pipeline method that receives `BaseDiscoveryProductContext` (or `ExploreContext`) can call `context.rankingSignalCollector.record()` without casting or special handling. Non homepage contexts like notifications and category filters get the no op default and pay zero cost.

The collector is a mutable object held by reference on the immutable `ExploreContext` data class. When `ExploreContext` is copied (which happens 4 to 8 times per request), the reference is preserved. Signals recorded early in the pipeline are readable later.

### Supported Carousel Types

The framework works for every carousel type on the homepage because `ExploreContext` is passed through every path:

| Carousel Type | Collection Model | Adapter | Supported |
|---|---|---|---|
| Store (all variants) | `LiteStoreCollection` | `StoreCarouselDataAdapter` | Yes (wired in this RFC) |
| Deals | `DealCarousel` | `DealCarouselDataAdapter` | Yes (one line addition when needed) |
| Items | `ItemCarousel` | `ItemCarouselPageDataAdapter` | Yes (one line addition when needed) |
| Announcements/Reels | `AnnouncementCollection` | `AnnouncementsAdapter` | Yes (one line addition when needed) |

Initial scope wires `StoreCarouselDataAdapter` only. Other adapters require adding one `RankingSignalWriter` call each.

### Context Availability

This framework requires `BaseDiscoveryProductContext` to be accessible where signals are recorded. We validated this against every signal currently logged in `SCORE_MODIFIERS` (see [research](research-explore-context-availability.md) for full trace).

`ExploreContext` is available at every place where scores are currently computed, transferred, or logged in the homepage pipeline. This includes the ranking phase (`rankAndMergeContent`), the decoration phase (`updateStoreEntity`), the GenAI reranking step (`reRankStoresByScoreAndSimilarity`), and every adapter that builds a logging map.

The one place it is intentionally not available is inside the deepest scorer implementations like `ContextualStoreScorer` and `SibylRegressor`. These are pure computation functions that receive narrow, scorer specific context types. They should not be responsible for logging. The right pattern is: the scorer computes the value, and the **caller** that invoked the scorer and holds pipeline context records it. This is the same separation of concerns that already exists in the codebase. The scorer does not know about Iguazu or Snowflake. The pipeline orchestration layer does. The framework reinforces this boundary rather than breaking it.

## Adding a New Signal After This Ships

This is the target experience. An engineer working on a ranking feature wants to log a new signal to Snowflake.

**Before (current state):**
1. Add a typed field to a data class
2. Copy it through carrier objects during decoration
3. Add a hardcoded adapter mapping
4. Add a `LoggedValue` enum entry
5. 3 to 4 files changed, multiple code owners to review

**After (with this framework):**
```kotlin
context.rankingSignalCollector.record(carousel.id, store.storeId(), "my_new_signal", value)
```
1 line, 1 file, the signal appears in Snowflake automatically.

## Files Changed

**New** (2):
- `RankingSignalCollector.kt`: interface, `DefaultRankingSignalCollector`, `NOOP` singleton
- `RankingSignalWriter.kt`: static utility with prefix logic and type detection

**Modified** (5):
- `BaseDiscoveryContext.kt`: add `rankingSignalCollector` with no op default to `BaseDiscoveryProductContext` interface
- `ExploreContext.kt`: override with `DefaultRankingSignalCollector()`
- `events.proto`: add `ranking_signals` and `carousel_ranking_signals` map fields to `CrossVerticalHomePageFeedEvent`
- `StoreCarouselDataAdapter.kt`: one line `RankingSignalWriter` call in `generateStoreLogging()`
- `ContainerEventsGenerator.kt`: prefix based forwarding loop after `LoggedValue.assign()`

## Timeline

3 to 4 days. 2 new files, 5 modified files, 1 proto change. Unit tests for the collector, writer, adapter, and event generator. Sandbox validation to confirm signals land in Snowflake.

## Open

- **Iguazu column auto creation**: Verify that the new proto map fields automatically create `RANKING_SIGNALS` and `CAROUSEL_RANKING_SIGNALS` VARIANT columns in Snowflake, or identify the manual steps needed.
