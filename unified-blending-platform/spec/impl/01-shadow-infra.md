# Part 1: Shadow Infrastructure

## Goal

Run both the old ranking path and the new UBP engine path on every request simultaneously.
Always return the old result to the user. Log any sort-order differences as structured errors
for comparison. This validates that `control.json` exactly replicates prod behavior before any
real traffic migrates.

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
  └── if divergence → logger.warn(structured diff)
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

            val oldOrder = oldResult.first.toSortOrderMap()
            val newOrder = newResult.toSortOrderMap()
            val divergences = oldOrder.filter { (id, pos) -> newOrder[id] != pos }

            if (divergences.isNotEmpty()) {
                logger.warn(
                    "ubp_shadow_divergence layer=vertical " +
                    "request_id=${context.requestId} " +
                    "divergence_count=${divergences.size} " +
                    "diffs=${divergences.map { (id, old) -> "$id:$old->${newOrder[id]}" }.joinToString(",")}"
                )
            }
        }.onFailure { logger.warn("ubp_shadow_error layer=vertical request_id=${context.requestId}", it) }

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

                val divergences = oldResult.storeIds
                    .zip(newResult.storeIds)
                    .filter { (old, new) -> old != new }

                if (divergences.isNotEmpty()) {
                    logger.warn(
                        "ubp_shadow_divergence layer=horizontal " +
                        "request_id=${context.requestId} " +
                        "carousel_id=${collection.id} " +
                        "divergence_count=${divergences.size}"
                    )
                }
            }.onFailure { logger.warn("ubp_shadow_error layer=horizontal request_id=${context.requestId}", it) }
        }

        oldResult  // always return old
    }
}.awaitAll()
```

## Log Format

All shadow output uses a single log line with structured key=value pairs:

```
ubp_shadow_divergence  layer=vertical  request_id=<id>  divergence_count=<n>  diffs=<id>:<old>-><new>,...
ubp_shadow_error       layer=vertical  request_id=<id>  (exception attached)
```

This makes divergences grep-able and queryable in the logging system without any new infra.

**Validation target:** `divergence_count = 0` on sampled requests before promoting to real traffic.

## New DV Manifest Entries

```kotlin
// In DiscoveryExperimentManager.Manifest
UBP_HP_VERTICAL_SHADOW("ubp_hp_vertical_shadow"),
UBP_HP_HORIZONTAL_SHADOW("ubp_hp_horizontal_shadow"),
```

Both default to `"off"`. Enable `"shadow"` for a canary % of traffic.

## Critical Rules

1. **New path errors must never affect users.** Always wrapped in `runCatching`. On failure: log with `ubp_shadow_error`, skip, return old result.
2. **New path latency must not add to p99.** Log its duration; alert if it exceeds 5ms p99 additional.
3. **Shadow is temporary.** Delete the shadow branch once the control arm is validated. Don't let it become permanent scaffolding.

## Files to Create / Modify

| Action | File |
|---|---|
| Modify | `DefaultHomePagePostProcessor.kt` — add shadow branch |
| Modify | `DefaultHomePageStoreRanker.kt` — add shadow branch |
| Modify | `DiscoveryExperimentManager.kt` — add 2 DV entries |

No new classes, no new protos. Just the two modified files and the DV entries.

## Prerequisites

None. This is the first thing built. The `FeedRowRanker` reference in the shadow will be a
stub/no-op until Parts 2–5 are complete. The shadow DV stays `"off"` until the engine is real.

## Done When

- Shadow runs on canary traffic (`ubp_hp_vertical_shadow = "shadow"` for 1% of users)
- Logs show `ubp_shadow_divergence` lines with `divergence_count=0` for vertical
- Latency impact < 5ms p99 additional
