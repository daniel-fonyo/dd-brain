# Part 1: Shadow Infrastructure

## Goal

Run both the old ranking path and the new UBP engine path on every request simultaneously.
Always return the old result to the user. Emit both outputs to Snowflake for comparison.
This validates that `control.json` exactly replicates prod behavior before any real traffic migrates.

## Design Pattern: Proxy (Structural)

> "Proxy provides a surrogate or placeholder for another object. A proxy controls access to the
> original object, allowing you to perform something either before or after the request gets
> through to the original object."
> — refactoring.guru

The shadow is a Proxy around the ranking call. It intercepts, runs both old and new,
returns the old result unchanged. Callers are unaware anything changed.

```
Caller
  │
  ▼
ShadowRankingProxy.rank()
  ├── oldPath() → result_old   (always returned)
  ├── newPath() → result_new   (run but discarded)
  └── emit(requestId, result_old, result_new) → Iguazu
```

## Is-a / Has-a

- `ShadowRankingProxy` IS NOT a `FeedRowRanker` — it exists only during the migration window
- `ShadowRankingProxy` HAS A reference to the old ranking logic (lambda/function) and HAS A `FeedRowRanker`
- Keep it as a simple wrapper function, not a full class — it will be deleted

## Vertical Shadow

**File:** `DefaultHomePagePostProcessor.kt`
**Incision point:** Inside `reOrderGlobalEntitiesV2()`, wrapping `rankAndDedupeContent()`

```kotlin
// Controlled by DV: ubp_hp_vertical_shadow (values: "off" | "shadow")
val shadowMode = discoveryExperimentManager.getUbpVerticalShadowMode(context.experimentMap)

val (content, placementEventData) = when (shadowMode) {
    "shadow" -> {
        // Run old path
        val oldResult = rankAndDedupeContent(context, ...)

        // Run new path — errors must never propagate to user
        runCatching {
            val rows = toFeedRows(layout)
            val newResult = feedRowRanker.rank(rows, "control", context)
            ubpShadowEmitter.emit(
                requestId = context.requestId,
                layer = "vertical",
                oldSortOrders = oldResult.first.toSortOrderMap(),
                newSortOrders = newResult.toSortOrderMap(),
            )
        }.onFailure { logger.warn("UBP shadow vertical failed", it) }

        oldResult  // always return old
    }
    else -> rankAndDedupeContent(context, ...)
}
```

## Horizontal Shadow

**File:** `DefaultHomePageStoreRanker.kt`
**Incision point:** Inside `rank()`, wrapping each `modifyLiteStoreCollection()` call

```kotlin
// Controlled by DV: ubp_hp_horizontal_shadow (values: "off" | "shadow")
val shadowMode = discoveryExptManager.getUbpHorizontalShadowMode(context.experimentMap)

return scoredCollections.map { collection ->
    CoroutineScope(...).async {
        val oldResult = modifyLiteStoreCollection(context, collection, storesById, metadata, ...)

        if (shadowMode == "shadow") {
            runCatching {
                val newResult = rowItemRanker.rank(collection, "control", context)
                ubpShadowEmitter.emit(
                    requestId = context.requestId,
                    layer = "horizontal",
                    carouselId = collection.id,
                    oldStoreOrder = oldResult.storeIds,
                    newStoreOrder = newResult.storeIds,
                )
            }.onFailure { logger.warn("UBP shadow horizontal failed", it) }
        }

        oldResult  // always return old
    }
}.awaitAll()
```

## Shadow Emitter

**New class:** `UbpShadowEmitter`

```kotlin
class UbpShadowEmitter(private val iguazuModule: IguazuModule) {
    fun emit(requestId: String, layer: String, oldSortOrders: Map<String, Int>, newSortOrders: Map<String, Int>) {
        // Emit one event per divergent row_id
        val divergences = oldSortOrders.entries
            .filter { (id, oldPos) -> newSortOrders[id] != oldPos }

        if (divergences.isNotEmpty()) {
            iguazuModule.emit(UbpShadowDivergenceEvent(
                requestId = requestId,
                layer = layer,
                divergences = divergences.map { (id, oldPos) ->
                    SortOrderDivergence(rowId = id, oldPosition = oldPos, newPosition = newSortOrders[id])
                }
            ))
        }
    }
}
```

Emit only on divergence — not on every request. Keeps Iguazu volume manageable.

## New DV Manifest Entries

```kotlin
// In DiscoveryExperimentManager.Manifest
UBP_HP_VERTICAL_SHADOW("ubp_hp_vertical_shadow"),
UBP_HP_HORIZONTAL_SHADOW("ubp_hp_horizontal_shadow"),
```

Both default to `"off"`. Enable `"shadow"` for a canary % of traffic.

## Snowflake Validation Query

```sql
SELECT
    layer,
    COUNT(*) AS total_requests,
    COUNTIF(divergence_count > 0) AS divergent_requests,
    COUNTIF(divergence_count > 0) / COUNT(*) AS divergence_rate,
    AVG(divergence_count) AS avg_divergences_per_request
FROM ubp_shadow_divergence_events
WHERE _date >= CURRENT_DATE - 1
GROUP BY layer
ORDER BY layer;
```

Target: divergence_rate < 0.01 (>99% match) before promoting to real traffic.

## Critical Rules

1. **New path errors must never affect users.** Always wrapped in `runCatching`. On failure: log, skip, return old result.
2. **New path latency must not add to p99.** Run new path asynchronously if it threatens latency budget. Log its duration separately.
3. **Shadow is temporary.** Delete the shadow branch once Stage 2 (real traffic) is validated. Don't let it become permanent scaffolding.

## Files to Create / Modify

| Action | File |
|---|---|
| Modify | `DefaultHomePagePostProcessor.kt` — add shadow branch |
| Modify | `DefaultHomePageStoreRanker.kt` — add shadow branch |
| New | `libraries/common/.../ubp/shadow/UbpShadowEmitter.kt` |
| New | proto: `ubp_shadow_divergence.proto` |
| Modify | `DiscoveryExperimentManager.kt` — add 2 DV entries |

## Prerequisites

None. This is the first thing built. The `FeedRowRanker` reference in the shadow will be a
stub/no-op until Parts 2–5 are complete. The shadow DV stays `"off"` until the engine is real.

## Done When

- Shadow runs on canary traffic (`ubp_hp_vertical_shadow = "shadow"` for 1% of users)
- Snowflake query shows divergence_rate < 0.01 for vertical
- Latency impact < 5ms p99 additional
