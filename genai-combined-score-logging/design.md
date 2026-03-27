# Design: RankingSignalCollector — Universal Ranking Signal Logging

**Date**: 2026-03-27
**Status**: Draft
**Scope**: Generic infrastructure for logging per-entity and per-carousel ranking signals to Snowflake, across all homepage carousel types.
**First consumer**: GenAI `combinedScore` logging.

## Problem

Ranking decisions made during the homepage pipeline — scores, weights, model names, strategies — are **discarded after use**. Only a subset of signals survive to Snowflake, and those that do require per-field wiring through typed fields on `StoreEntity`, hardcoded adapter mappings, and `LoggedValue` enum entries. This makes adding new signals expensive and error-prone.

The immediate need is logging GenAI `combinedScore`, but the same gap exists for any ranking signal across any carousel type.

## Design Goals

1. **Universal**: Works for all homepage carousel types (store, item, deal, announcement), not just GenAI or store carousels.
2. **Zero ceremony for callers**: Any pipeline step records a signal with one call — no typed fields, no adapter wiring, no constants.
3. **Separation of concerns**: Signal producers don't know about logging plumbing. The infrastructure handles collection, forwarding, and emission.
4. **Mixed types**: Supports doubles (scores, weights) and strings (model names, strategies).
5. **Both levels**: Store-level signals (vary per entity in a carousel) and carousel-level signals (same for all entities).

## Key Decisions

### 1. Standalone collector on ExploreContext, not a field on collection types

**Decision**: `RankingSignalCollector` is a standalone mutable object living on `ExploreContext`.

**Why not on LiteStoreCollection?**
- `LiteStoreCollection` is immutable (data class), copied 4-8 times per request. A mutable collector inside an immutable class is a code smell, and an immutable map means verbose `.copy()` chains for every signal.
- More importantly, `LiteStoreCollection` only covers store carousels. Item carousels (`ItemCarousel`), deal carousels (`DealCarousel`), reels (`AnnouncementCollection`) use completely different models with **no shared base class**. The only common interface is `SortablePlacement` (just `sortOrder`).
- `ExploreContext` is already threaded through the entire pipeline for all carousel types. It's the one convergence point.

**Why not on entity types (StoreEntity, etc.)?**
- Each carousel type has its own entity (`StoreEntity`, `ItemStoreEntity`, `Deal`, `AnnouncementEntity`) with no shared hierarchy.
- Adding `rankingSignals: Map<String, String>` to each entity duplicates the pattern without a shared contract.
- The collector-to-adapter path can bypass entities entirely (see RankingSignalWriter below).

**Alternatives considered**:
- Interface `RankingSignalCarrier` implemented by all collection types → too invasive, 6+ data classes, no existing shared contract to hook into.
- Map field on `LiteStoreCollection` with extension function → works for store carousels only, not universal.

### 2. RankingSignalWriter utility, not per-adapter polymorphism

**Decision**: A standalone `RankingSignalWriter` utility that any adapter calls with one line.

**Why not a shared adapter interface?**
The homepage adapters have **no shared base class or interface** for cross-type dispatch:

| Adapter | Input Type | Output Type | Interface |
|---|---|---|---|
| `StoreCarouselDataAdapter` | `StoreCarousel` | `FacetV2` | `StoreCarouselAdapterInterface` |
| `DealCarouselDataAdapter` | `DealCarousel` | `FacetV2` / `FacetSection` | None |
| `ItemCarouselPageDataAdapter` | `ItemCarousel` | `FacetSection` | None |
| `AnnouncementsAdapter` | `AnnouncementCollection` | `MxCrmAnnouncement` | None |

Different input types, different output types, different method signatures. Unifying them would require a major refactor unrelated to our goal.

**What they DO share**: Every adapter builds a `MutableMap<String, Value>` logging map and produces a `Struct`. The writer hooks into this shared step.

### 3. Proto `map<string, string>` for the Snowflake column

**Decision**: Add `map<string, string> ranking_signals = <next_id>` to `CrossVerticalHomePageFeedEvent`.

**Why `map<string, string>` (not `map<string, double>`)?**
- We need to log both doubles (scores, weights) and strings (model names, strategies).
- Snowflake handles casting transparently: `RANKING_SIGNALS:genai_combined_score::DOUBLE` works on string values that parse as numbers.
- One map, one column, no type bifurcation.

**Why not `google.protobuf.Struct`?**
- Heavier proto import, no practical benefit over `map<string, string>` for flat key-value pairs.

**Why not reuse existing `score_modifiers` (repeated ScoreModifier)?**
- `score_modifiers` is a JSON **array** requiring expensive `LATERAL FLATTEN` in Snowflake.
- It's consumed by production ML jobs — changing its shape or semantics is risky.
- The new `map<string, string>` serializes as a JSON **object** with direct key access: `RANKING_SIGNALS:key::TYPE`. No flatten, no row explosion.

**Why a new column instead of packing into existing columns?**
- `CAROUSEL_DETAILS` is a serialized JSON string blob, not a structured proto field.
- `SCORE_MODIFIERS` has the array/flatten problem described above.
- A new proto map field → new VARIANT column in Snowflake → clean, queryable, independent.

### 4. Naming: `rankingSignals` not `scoreModifiers`, `rankingMetrics`, or `rankingContext`

**Decision**: Use `rankingSignals` throughout.

| Rejected name | Why |
|---|---|
| `scoreModifiers` | Shadows existing proto `repeated ScoreModifier score_modifiers` field (field 59) |
| `storeRankingMetrics` | "Metrics" implies only numbers. We also log strings (model names, strategies). |
| `rankingContext` | Name collision — `RankingContext` already exists in `sdk-dex` as the input context for ranking (consumer/geo/experiment info). Completely different concept. |
| `rankingMetadata` | Name collision — `rankingMetadata: Map<String, Any?>` already exists on `StoreEntity` for search pipeline debugging. Never read by homepage adapters. |
| `rankingDecisions` | Reasonable but less standard than "signals" in ML/ranking terminology. |

`rankingSignals` is unambiguous: the signals (inputs and outputs) of ranking decisions.

### 5. No typed intermediary fields

**Decision**: Callers write directly to the collector map. No `GeneratedRecommendationStoreInfo.combinedScore`, no `StoreEntity.genaiCombinedScore`, no new typed fields anywhere.

**Why?**
- Every typed field requires: (1) add field to data class, (2) manually copy to next carrier, (3) add adapter mapping, (4) add LoggedValue enum entry. This is the pattern that made `StoreEntity` grow to 226 fields.
- The collector map IS the extensible container. Adding a new signal is one `record()` call at the source. No downstream changes.

### 6. Signals bypass entity types entirely

**Decision**: `RankingSignalWriter` reads from the collector and writes directly into the adapter's logging map. Signals do not pass through `StoreEntity`, `Deal`, `ItemStoreEntity`, or any entity class.

**Flow**:
```
Pipeline step → collector.record(carouselId, entityId, key, value)
                                    ↓
Adapter → RankingSignalWriter.writeToLogging(collector, carouselId, entityId, loggingMap)
                                    ↓
Existing logging infrastructure → proto event → Iguazu → Snowflake
```

No entity class changes. No decoration step changes. The collector and writer are the only new code.

## Architecture

### Components

```
┌──────────────────────────────────────────────────────────┐
│ ExploreContext (request-scoped)                           │
│  └─ rankingSignalCollector: RankingSignalCollector        │
└──────────────────────────────────────────────────────────┘
         │ .record()                        │ .forEntity() / .forCarousel()
         ▼                                  ▼
┌─────────────────────┐          ┌──────────────────────────┐
│ Pipeline Steps       │          │ RankingSignalWriter       │
│ (any carousel type)  │          │ (static utility)          │
│                      │          │                            │
│ GenAI reranker       │          │ writeToLogging(            │
│ Taste ranker         │          │   collector, carouselId,   │
│ Unified ranker       │          │   entityId, loggingMap)    │
│ Deal scorer          │          │                            │
│ Item ranker          │          │ writeCarouselSignals(      │
│ ...                  │          │   collector, carouselId,   │
└─────────────────────┘          │   loggingMap)              │
                                  └──────────────────────────┘
                                           │
                                           ▼
                                  ┌──────────────────────────┐
                                  │ Existing Logging Pipeline  │
                                  │                            │
                                  │ loggingMap → Struct →      │
                                  │ ContainerEventsGenerator → │
                                  │ proto event → Iguazu →     │
                                  │ Snowflake                  │
                                  └──────────────────────────┘
```

### RankingSignalCollector

```kotlin
/**
 * Request-scoped collector for ranking signals.
 * Lives on ExploreContext. Any pipeline step can record signals;
 * adapters consume them via RankingSignalWriter.
 *
 * Thread-safe: homepage pipeline uses coroutines with parallel carousel processing.
 */
class RankingSignalCollector {
    // Entity-level signals: carousel → entity → key/value
    private val entitySignals = ConcurrentHashMap<String, ConcurrentHashMap<Long, ConcurrentHashMap<String, String>>>()

    // Carousel-level signals: carousel → key/value
    private val carouselSignals = ConcurrentHashMap<String, ConcurrentHashMap<String, String>>()

    /** Record a per-entity signal (e.g., a store's combined score). */
    fun record(carouselId: String, entityId: Long, key: String, value: Any) {
        entitySignals
            .getOrPut(carouselId) { ConcurrentHashMap() }
            .getOrPut(entityId) { ConcurrentHashMap() }[key] = value.toString()
    }

    /** Record a per-carousel signal (e.g., reranker alpha/beta). */
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

**Thread safety note**: The homepage pipeline processes carousels in parallel via `async/awaitAll`. Different carousels write to different `carouselId` keys so contention is low, but `ConcurrentHashMap` prevents race conditions on the outer maps.

### RankingSignalWriter

```kotlin
/**
 * Utility to flush ranking signals from the collector into an adapter's logging map.
 * Called by each adapter with one line — no per-signal wiring needed.
 */
object RankingSignalWriter {

    /**
     * Write all entity-level signals for the given carousel+entity into the logging map.
     * Numeric-parseable values are written as NumberValue, others as StringValue.
     */
    fun writeEntitySignals(
        collector: RankingSignalCollector,
        carouselId: String,
        entityId: Long,
        map: MutableMap<String, Value>,
    ) {
        collector.forEntity(carouselId, entityId).forEach { (key, value) ->
            val doubleVal = value.toDoubleOrNull()
            map[key] = if (doubleVal != null) {
                AdapterUtil.getSafeNumberValueWithDefault(doubleVal)
            } else {
                AdapterUtil.getStringValue(value)
            }
        }
    }

    /**
     * Write all carousel-level signals into the logging map.
     */
    fun writeCarouselSignals(
        collector: RankingSignalCollector,
        carouselId: String,
        map: MutableMap<String, Value>,
    ) {
        collector.forCarousel(carouselId).forEach { (key, value) ->
            val doubleVal = value.toDoubleOrNull()
            map[key] = if (doubleVal != null) {
                AdapterUtil.getSafeNumberValueWithDefault(doubleVal)
            } else {
                AdapterUtil.getStringValue(value)
            }
        }
    }
}
```

### Proto Change

In `services-protobuf/protos/feed_service/events.proto`, on `CrossVerticalHomePageFeedEvent`:

```protobuf
// Per-entity ranking signals as a flat key-value map.
// Queryable in Snowflake as RANKING_SIGNALS:key_name::DOUBLE or ::VARCHAR.
// Populated by RankingSignalCollector via RankingSignalWriter in feed-service.
// Existing score_modifiers (field 59) is unchanged.
map<string, string> ranking_signals = 86;
//next id: 87
```

### ContainerEventsGenerator Change

After `LoggedValue.assign(storeLogging, this, STORE_LOGGED_VALUES)`, populate the proto map from unhandled logging map entries:

```kotlin
val handledKeys = currentLoggedValues.map { it.key }.toSet()
storeLogging.forEach { (key, value) ->
    if (key !in handledKeys) {
        when (this) {
            is Events.CrossVerticalHomePageFeedEvent.Builder ->
                putRankingSignals(key, if (value.hasNumberValue()) value.numberValue.toString() else value.stringValue)
        }
    }
}
```

This is the generic bridge: any key in the logging map that isn't already handled by a typed `LoggedValue` gets forwarded to the `ranking_signals` proto map. No per-signal wiring.

## Snowflake Access

After the proto change deploys and Iguazu picks up the new field:

```sql
-- Direct key access (no FLATTEN):
SELECT
    RANKING_SIGNALS:genai_combined_score::DOUBLE AS genai_combined_score,
    RANKING_SIGNALS:ranking_model::VARCHAR AS ranking_model,
    STORE_ID,
    FACET_NAME
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
WHERE IGUAZU_PARTITION_DATE = CURRENT_DATE()
  AND IGUAZU_PARTITION_HOUR = HOUR(CURRENT_TIMESTAMP()) - 1
  AND RANKING_SIGNALS IS NOT NULL
```

**Note**: The actual table name is `CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE` (with underscores and `_ICE` suffix for Iceberg format). The `RANKING_SIGNALS` column will appear as a VARIANT type. Verify after proto merge that Iguazu auto-creates it.

## Homepage Pipeline — Full Context

### Order of Operations (All Carousel Types)

```
PHASE 1: CREATION (varies by type)
  ├─ GenAI: HomepageGeneratedRecommendationProduct.buildCollection() → LiteStoreCollection
  ├─ Taste: DynamicTasteCarouselService → LiteStoreCollection
  ├─ Campaign: PromoService → LiteStoreCollection
  ├─ Sponsored: AdStoreCarousels → LiteStoreCollection(isSponsored=true)
  ├─ Deals: DealService → DealCarousel
  ├─ Items: ItemCarouselService → ItemCarousel
  └─ Reels: MxCRM SDK → AnnouncementCollection

PHASE 2: TYPE-SPECIFIC TRANSFORMS (where signals are produced)
  ├─ GenAI: reRankStoresByScoreAndSimilarity() ← collector.record() HERE
  ├─ Taste: filtering/scoring ← collector.record() HERE
  ├─ PromoGating: eligibility ← collector.record() HERE
  └─ Most types: no type-specific transform

PHASE 3: UNIFIED RANKING (all LiteStoreCollections)
  DefaultHomePagePostProcessor.rankAndDedupeContent()
  → EntityRankerConfiguration 5-step ranking
  ← collector.record() HERE (for unified ranker signals)

PHASE 4: LAYOUT
  DefaultHomePageStoreLayoutProcessor.constructWorkflow()

PHASE 5: MATERIALIZATION (collection → carousel + entity objects)
  ├─ Store: StoreCarouselService.generateSingleStoreCarousel() → StoreCarousel w/ StoreEntity
  ├─ Deals: already DealCarousel w/ Deal objects
  ├─ Items: already ItemCarousel w/ ItemStoreEntity objects
  └─ Reels: already AnnouncementCollection w/ AnnouncementEntity objects

PHASE 6: SERIALIZATION
  Each type → its own adapter → logging map
  ← RankingSignalWriter.writeEntitySignals() HERE (one line per adapter)
  ← RankingSignalWriter.writeCarouselSignals() HERE

PHASE 7: IGUAZU (async, fire-and-forget)
  ContainerEventsGenerator → proto events → Snowflake
  ← Unhandled logging map keys → ranking_signals proto map
```

### Carousel Types and Their Paths

| Type | Collection Model | Entity | Adapter | Collector works? |
|---|---|---|---|---|
| Store (all flavors) | `LiteStoreCollection` | `StoreEntity` | `StoreCarouselDataAdapter` | Yes |
| Items | `ItemCarousel` | `ItemStoreEntity` | `ItemCarouselPageDataAdapter` | Yes |
| Deals | `DealCarousel` | `Deal` | `DealCarouselDataAdapter` | Yes |
| Reels | `AnnouncementCollection` | `AnnouncementEntity` | `AnnouncementsAdapter` | Yes |
| Collections | `Collection` / `CollectionV2` | Nested `StoreCarousel` | `CollectionStandardDataAdapter` | Yes |
| Banners | `BannerCarousel` | `BannerContent` | Unknown/minimal | Low priority |

All types except banners have an adapter that builds a logging map. All have access to `ExploreContext`. The collector + writer pattern works for all of them.

## What Exists Today (Relevant Fields)

### On StoreEntity (226 fields)
| Field | Type | Used by homepage logging? | Notes |
|---|---|---|---|
| `rankingMetadata` | `Map<String, Any?>` | **No** — only search pipeline | Populated by `SearchBlenderServiceRepository`, read by `SearchStoreRowMapper`. Dead for homepage. |
| `dynamicMerchantTags` | `Map<String, String>` | No | Merchant-specific tags |
| 15+ typed score fields | `Double?` each | Yes — manual field-by-field | `predictionScore`, `srMultiplier`, `genaiEmbeddingSimilarityScore`, etc. |

### On LiteStoreCollection
| Field | Type | Notes |
|---|---|---|
| `trackingPayload` | `Map<String, Any>` | Carousel-level freeform metadata. Serialized as JSON string into `CAROUSEL_DETAILS` logging key. |
| `storePredictionScoresMap` | `Map<Long, ScoreWithFactors>` | Unified ranker scores per store. |
| `generatedRecommendationStoreInfoMap` | `Map<Long, GeneratedRecommendationStoreInfo>` | GenAI-specific per-store data. |

### Existing Snowflake Columns (CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE)
| Column | Type | Source |
|---|---|---|
| `HORIZONTAL_ELEMENT_SCORE` | DOUBLE | `storePredictionScoresMap[id].finalScore` |
| `SCORE_MODIFIERS` | VARIANT (JSON array) | `repeated ScoreModifier` proto — requires FLATTEN |
| `FACET_SCORE` / `RAW_FACET_SCORE` | DOUBLE | Facet-level scores |
| `RANKING_SIGNALS` | **Does not exist yet** | Will be created by proto change |

## Files to Create/Modify

### New Files (feed-service)
| File | Description |
|---|---|
| `libraries/platform/.../ranking/RankingSignalCollector.kt` | Collector class |
| `libraries/domain-util/.../utils/logging/RankingSignalWriter.kt` | Writer utility |

### Modified Files (feed-service)
| File | Change |
|---|---|
| `libraries/platform/.../models/ExploreContext.kt` (or equivalent) | Add `val rankingSignalCollector: RankingSignalCollector = RankingSignalCollector()` |
| `libraries/domain-util/.../facets/adapters/StoreCarouselDataAdapter.kt` | One-line call to `RankingSignalWriter.writeEntitySignals()` in logging method |
| `libraries/domain-util/.../iguazu/ContainerEventsGenerator.kt` | Generic loop: unhandled logging map keys → `ranking_signals` proto map |

### Modified Files (services-protobuf)
| File | Change |
|---|---|
| `protos/feed_service/events.proto` | Add `map<string, string> ranking_signals = 86` to `CrossVerticalHomePageFeedEvent` |

### Future Adapter Additions (not in scope now)
| File | Change |
|---|---|
| `DealCarouselDataAdapter.kt` | One-line `RankingSignalWriter` call |
| `ItemCarouselPageDataAdapter.kt` | One-line `RankingSignalWriter` call |
| `AnnouncementsAdapter.kt` | One-line `RankingSignalWriter` call |

## Open Questions

1. **Iguazu column auto-creation**: Does Iguazu automatically create a `RANKING_SIGNALS` VARIANT column when it sees the new proto map field? Or is a manual schema migration needed? Must verify before end-to-end validation.

2. **ExploreContext location**: Need to confirm the exact class/field where the collector lives. `ExploreContext` is the main candidate but there may be a more appropriate request-scoped object.

3. **Carousel-level signals proto field**: The current proto change only adds an entity-level `ranking_signals` field. Carousel-level signals (alpha, beta) currently go through `trackingPayload` → `CAROUSEL_DETAILS`. Do we also want a structured `carousel_ranking_signals` proto field, or is `CAROUSEL_DETAILS` sufficient for now?

4. **Other event types**: The proto change targets `CrossVerticalHomePageFeedEvent`. Other event types (`HomePageFeedEvent`, `VerticalPageFeedEvent`) may need the same field if ranking signals should be logged on non-cross-vertical pages.

## Related
- `brain/genai-combined-score-logging/analysis.md` — Original gap analysis for combinedScore
- `brain/genai-reranker-logging/` — PR #62113 adds embeddingScore + alpha/beta logging
- `brain/ranking-pipeline-refactor.md` — Broader RFC for ranking pipeline refactor (strangler fig)
