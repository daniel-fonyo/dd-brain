# Option A: Unified Carousel Positioning — Implementation Plan

**Date:** 2026-03-30
**Prereq:** Parul's SOT audit + config cleanup complete
**Approach:** Standalone, before RFC. DV-gated with A/A shadow validation.

---

## New Files (3)

### File 1: `CarouselPositioningConfig.kt`

New data model + runtime reader for the unified config.

```
libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/ranking/CarouselPositioningConfig.kt
```

```kotlin
package com.doordash.consumer.feed.domainUtil.ranking

import com.doordash.consumer.feed.platform.utils.PlatformRuntimeUtil.FEED_RUNTIME
import com.fasterxml.jackson.core.type.TypeReference

data class CarouselPositionRule(
    val position: Int,
)

data class CarouselPositioningConfig(
    val carousels: Map<String, CarouselPositionRule> = emptyMap(),  // exact ID → rule
    val prefixRules: Map<String, CarouselPositionRule> = emptyMap(),  // prefix → rule (for "taste:", "rt:")
) {
    /**
     * Find the position rule for a carousel ID.
     * Exact match takes priority over prefix match.
     */
    fun findRule(carouselId: String): CarouselPositionRule? {
        // Exact match first
        carousels[carouselId]?.let { return it }

        // Prefix match (e.g., "taste:" matches "taste:pizza", "rt:" matches "rt:abc-123")
        for ((prefix, rule) in prefixRules) {
            if (carouselId.startsWith(prefix)) return rule
        }
        return null
    }

    companion object {
        private const val CONFIG_PATH = "carousels/carousel_positioning_config.json"
        private val CONFIG_TYPE = object : TypeReference<CarouselPositioningConfig>() {}

        fun load(): CarouselPositioningConfig {
            return FEED_RUNTIME.getJson(CONFIG_PATH, CONFIG_TYPE, CarouselPositioningConfig())
        }
    }
}
```

### File 2: `carousel_positioning_config.json`

Runtime JSON config. Populated mechanically from existing `pinned_carousel_ranking_order.json` + `boost_by_position_allow_list.json` + Parul's SOT.

```
Runtime: carousels/carousel_positioning_config.json
```

```json
{
  "carousels": {
    "d4f2a1b3-...": { "position": 0 },
    "a1b2c3d4-...": { "position": 1 },
    "7e8f9a0b-...": { "position": 5 }
  },
  "prefixRules": {
    "taste:": { "position": 2 },
    "rt:": { "position": 3 }
  }
}
```

### File 3: `UnifiedPositioningShadow.kt`

Shadow comparison helper. Emits metrics comparing old vs new path output.

```
libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/ranking/UnifiedPositioningShadow.kt
```

```kotlin
package com.doordash.consumer.feed.domainUtil.ranking

import com.doordash.consumer.feed.domainUtil.ranking.models.RankableContent

object UnifiedPositioningShadow {
    /**
     * Compare two RankableContent outputs and emit parity metrics.
     * Returns true if outputs match.
     */
    fun compareAndEmit(old: RankableContent, new: RankableContent): Boolean {
        val oldOrder = extractSortOrderMap(old)
        val newOrder = extractSortOrderMap(new)

        val sortOrderMatch = oldOrder == newOrder
        val carouselCountMatch = oldOrder.size == newOrder.size

        // Per-carousel position deltas
        val deltas = (oldOrder.keys + newOrder.keys).distinct().map { id ->
            val oldPos = oldOrder[id]
            val newPos = newOrder[id]
            id to when {
                oldPos == null -> "new_only"
                newPos == null -> "old_only"
                else -> (newPos - oldPos).toString()
            }
        }

        // Emit metrics via p13nMetrics or Micrometer
        // ranking.unified_positioning.shadow.sortorder_match = sortOrderMatch
        // ranking.unified_positioning.shadow.carousel_count_match = carouselCountMatch
        // ranking.unified_positioning.shadow.position_deltas = deltas

        return sortOrderMatch
    }

    private fun extractSortOrderMap(content: RankableContent): Map<String, Int> {
        val map = mutableMapOf<String, Int>()
        content.storeCarousels.forEach { it.sortOrder?.let { so -> map[it.id] = so } }
        content.itemCarousels.forEach { map[it.id] = it.sortOrder }
        content.storeCollections.forEach { map[it.id] = it.sortOrder }
        content.collectionsV2.forEach { map[it.id] = it.sortOrder }
        content.dealCarousel?.let { it.sortOrder?.let { so -> map[it.id] = so } }
        content.mapCarousels.forEach { it.sortOrder?.let { so -> map[it.id] = so } }
        content.reelsCarousel?.let { it.sortOrder?.let { so -> map["reels"] = so } }
        return map
    }
}
```

---

## Modified Files (2)

### Mod 1: `DefaultHomePagePostProcessor.rankAndMergeContent()` — L708-759

This is the orchestration method. Today it has two branches (score-before-split vs split-before-score), both calling `splitPinnedAndRankCarousels()` + `mergeContentWithPushedSortOrder()`. The new path sends everything through ranking with the unified config.

**Current code** (`DefaultHomePagePostProcessor.kt` L708-759):

```kotlin
internal suspend fun rankAndMergeContent(context: ExploreContext, content: RankableContent): RankableContent {
    val (pinnedContent, rankedContent) = if (
        DiscoveryExperimentManager.enablePinnedCarouselsRankScores(context.experimentMap)
    ) {
        val rankedWholeContent = rankContent(context, content)
        val (pinnedRankedContent, nonPinnedRankedContent) = splitPinnedAndRankCarousels(context, rankedWholeContent)
        // ... build allNonPinnedContent ...
        pinnedRankedContent to allNonPinnedContent
    } else {
        val (originalPinnedContent, rankableCarousels) = splitPinnedAndRankCarousels(context, content)
        // ... rank only rankable ...
        originalPinnedContent to rankedRankableContent
    }
    return mergeContentWithPushedSortOrder(rankedContent, pinned...)
}
```

**New code:**

```kotlin
internal suspend fun rankAndMergeContent(context: ExploreContext, content: RankableContent): RankableContent {
    val useUnifiedPositioning = isUnifiedPositioningShadowOrServing(context.experimentMap)

    // OLD PATH — always runs during shadow, serves when not in serving treatment
    val oldResult = oldRankAndMergeContent(context, content)

    if (!useUnifiedPositioning) {
        return oldResult
    }

    // NEW PATH — rank ALL carousels, boost algorithm uses unified config for positioning
    val newResult = try {
        val contentCopy = content.deepCopy()  // isolate mutations
        rankContent(context, contentCopy)      // all carousels through ML + unified boost
    } catch (e: Throwable) {
        logger.error("unified_positioning_error", e)
        return oldResult  // shadow never impacts prod
    }

    // SHADOW: compare and emit, serve old
    if (isUnifiedPositioningShadow(context.experimentMap)) {
        UnifiedPositioningShadow.compareAndEmit(oldResult, newResult)
        return oldResult
    }

    // SERVING: new path serves
    return newResult
}

// Extracted: today's logic, unchanged
private suspend fun oldRankAndMergeContent(context: ExploreContext, content: RankableContent): RankableContent {
    // ... exact current code from L708-759, no changes ...
}
```

**What `deepCopy()` does:** The existing ranking code mutates `ScoreBundle` fields, `MutableSet` via `addNewToSet()`, and lists via `subList().clear()`. Both paths must operate on independent object graphs. This is the same requirement as the RFC's shadow mode.

### Mod 2: `BoostingBundle.boosted()` — L72-160

This is the 4-pass boost algorithm. Two changes:

**Change A: Replace allowlist + hinter with unified config (L89-132)**

```kotlin
// ===== CURRENT (L89-132) =====
val isTigerFeatureActive = TigerTeamUtil.isTigerTeamFeatureEnabled(...)
val boostByPositionAllowList = if (isTigerFeatureActive) {
    storeCarousels.filter { it.id.startsWith("dt:") }.map { it.id }.toSet()
} else {
    (PersonalizationRuntimeUtil.boostByPositionCarouselIdAllowList() +
        impressionCappedBoostingCarouselIds).toSet()
}
listOf(...all candidates...).flatten().forEach {
    if (isBoostingEnabled(boostByPositionAllowList, it.id()) || isShoppingTabPageType) {
        when (val hint = hinter(it)) {
            is BoostingHint.Unboosted -> unBoostPairs.add(Pair(hint, it))
            is BoostingHint.Boosted -> toBoostPairs.add(Pair(hint, it))
        }
    } else {
        unBoostPairs.add(Pair(BoostingHint.Unboosted, it))
    }
}

// ===== NEW (when useUnifiedPositioning=true) =====
val positioningConfig = CarouselPositioningConfig.load()
listOf(...all candidates...).flatten().forEach { candidate ->
    val rule = positioningConfig.findRule(candidate.id())
    if (rule != null) {
        // Config says this carousel should be at this position — no CMS dependency
        toBoostPairs.add(Pair(BoostingHint.Boosted(position = rule.position), candidate))
    } else {
        // Not in config — ML ranks freely
        unBoostPairs.add(Pair(BoostingHint.Unboosted, candidate))
    }
}
```

**What this eliminates:**
- `enforceManualSortOrder` dependency — position comes from config, not CMS
- `getBoostingHint()` calls — the 9 different `BoostingCandidate` subclass implementations are bypassed
- `boostByPositionAllowList` — absorbed into the unified config
- `isBoostingEnabled()` — replaced by `findRule() != null`
- Tiger Team override — if `dt:` carousels need boosting, they're in the config

**Change B: Delete NV unpin block (L134-160)**

```kotlin
// ===== CURRENT (L134-160) =====
if (pageType?.isHomepage() == true && P13nExperimentManager.enableHomepageVerticalBlending(experimentMap)) {
    val eligibleForUnpin = toBoostPairs.filter { (_, candidate) ->
        val nvEligible = isEligibleForUnpin(candidate)
        val afmExempt = isExemptAfmCarousel(candidate, experimentMap)
        nvEligible && !afmExempt
    }
    toBoostPairs.removeAll(eligibleForUnpin)
    eligibleForUnpin.forEach { (_, candidate) ->
        unBoostPairs.add(Pair(BoostingHint.Unboosted, candidate))
    }
}

// ===== NEW =====
// DELETED. If a carousel is in the config, it's positioned. If not, it's ML-ranked.
// No vertical ID check. No AFM exemption. The config IS the authority.
```

**Full method signature change** — `boosted()` gets a new `useUnifiedPositioning` parameter:

```kotlin
fun boosted(
    hinter: BoostingHinter,
    pageType: PageType? = null,
    impressionCappedBoostingCarouselIds: List<String> = emptyList(),
    experimentMap: Map<String, String> = emptyMap(),
    useUnifiedPositioning: Boolean = false,  // NEW
): BoostingBundle {
    // ...
    if (useUnifiedPositioning) {
        // NEW PATH: read from CarouselPositioningConfig
        val positioningConfig = CarouselPositioningConfig.load()
        listOf(...).flatten().forEach { candidate ->
            val rule = positioningConfig.findRule(candidate.id())
            if (rule != null) {
                toBoostPairs.add(Pair(BoostingHint.Boosted(rule.position), candidate))
            } else {
                unBoostPairs.add(Pair(BoostingHint.Unboosted, candidate))
            }
        }
        // NO unpin block — config is authority
    } else {
        // OLD PATH: existing allowlist + hinter + unpin logic (unchanged)
        // ... current L89-160 ...
    }
    // Rest of boost algorithm (L162+) is UNCHANGED — 4-pass logic works the same
}
```

**The 4-pass algorithm (L198-331) is untouched.** It receives `toBoostPairs` and `unBoostPairs` and produces the final ordering. The only change is how those pairs are populated.

---

## Call chain showing where `useUnifiedPositioning` flows

```
DefaultHomePagePostProcessor.rankAndMergeContent()
  │ reads DV: hp_unified_positioning_shadow / _serving
  │
  ├─ OLD PATH: oldRankAndMergeContent()
  │    → splitPinnedAndRankCarousels()      ← pinned list
  │    → rankContent()
  │      → EntityRankerConfiguration.rank()
  │        → getBoostBundle()
  │          → BoostingBundle.boosted(useUnifiedPositioning=false)
  │            → old allowlist + hinter + unpin
  │    → mergeContentWithPushedSortOrder()  ← push rankable after pinned
  │
  └─ NEW PATH: rankContent(contentCopy)
       → EntityRankerConfiguration.rank()
         → getBoostBundle()
           → BoostingBundle.boosted(useUnifiedPositioning=true)
             → CarouselPositioningConfig.load()
             → findRule(id) → Boosted(position) or Unboosted
             → NO unpin, NO allowlist, NO enforceManualSortOrder
```

The new path skips `splitPinnedAndRankCarousels` entirely — all carousels enter `rankContent()`. The boost algorithm in `boosted()` reads positions from the unified config instead of from `getBoostingHint()`. The 4-pass algorithm places them exactly as it does for boosted carousels today.

---

## How `useUnifiedPositioning` reaches `boosted()`

The flag needs to pass through `EntityRankerConfiguration.getBoostBundle()` (L274-531).

**Change in `EntityRankerConfiguration.getBoostBundle()`** (L504-510):

```kotlin
// CURRENT (L504-510)
val result = if (shouldBoostByPosition) {
    toBoostBundle.boosted(
        { it.getBoostingHint(context) },
        context.pageType,
        impressionCappedBoostingIds,
        experimentMap = context.experimentMap,
    )
}

// NEW
val result = if (shouldBoostByPosition) {
    toBoostBundle.boosted(
        { it.getBoostingHint(context) },
        context.pageType,
        impressionCappedBoostingIds,
        experimentMap = context.experimentMap,
        useUnifiedPositioning = context.useUnifiedPositioning,  // passed from DHPP
    )
}
```

`useUnifiedPositioning` is set on `RankingContext` by `DefaultHomePagePostProcessor` when the DV is in treatment. It flows through the existing call chain without changing any intermediate signatures except adding one field to `RankingContext`.

---

## What's NOT changed

| Component | Status | Why |
|---|---|---|
| `BoostingBundle.boosted()` L162-331 (4-pass algorithm) | **Unchanged** | It works on `toBoostPairs`/`unBoostPairs` — we only changed how they're populated |
| `EntityRankerConfiguration.getScoreBundle()` | **Unchanged** | ML scoring is identical |
| `EntityRankerConfiguration.getRankingBundle()` | **Unchanged** | Sort order from boost result |
| `EntityRankerConfiguration.getRankableContent()` | **Unchanged** | Reassembly from ranked entities |
| `CarouselCapping` | **Unchanged** | Capping is "should this be hidden," separate from positioning |
| `ImpressionCappedBoostingService` | **Unchanged for now** | ICB carousels can be added to the unified config by ID. Title matching stays until separate migration. |
| Post-ranking fixups | **Unchanged** | NV post-checkout, PAD, member pricing stay as-is. These are Phase 2. |
| `NonRankableHomepageOrderingUtil` | **Unchanged** | Final re-indexing stays |
| All `getBoostingHint()` implementations | **Unchanged** | Old path still uses them. New path bypasses them. |

---

## PR Breakdown

| PR | What | Size | Review burden |
|---|---|---|---|
| **PR 1** | `CarouselPositioningConfig.kt` + `UnifiedPositioningShadow.kt` + runtime JSON | ~150 lines new | Low — data model + reader, no behavior change |
| **PR 2** | `BoostingBundle.boosted()` adds `useUnifiedPositioning` branch + `RankingContext` field | ~60 lines changed | Medium — core boost logic, but old path untouched |
| **PR 3** | `DefaultHomePagePostProcessor.rankAndMergeContent()` shadow/serving orchestration | ~80 lines changed | Medium — entry point, shadow infra |
| **PR 4** | DV setup + dashboard + shadow enable at 1% | Config only | Low |

Total new/changed: ~290 lines. The old path is preserved in full behind `useUnifiedPositioning=false`. Rollback = turn off DV.

---

## Shadow Validation Checklist

Before moving from shadow to serving:

- [ ] `sortorder_match = true` for 100% of shadow requests (±0 tolerance)
- [ ] `carousel_count_match = true` for 100% of shadow requests
- [ ] `store_dedup_match = true` (stores that survive dedup are identical)
- [ ] Position deltas histogram shows all zeros
- [ ] Latency delta < 5ms p99 (config read + skip split should be faster, not slower)
- [ ] No error logs from shadow path
- [ ] Dashboard reviewed for 48+ hours at 3% traffic

---

## Risks and Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Dedup divergence (different stores survive because positions differ slightly) | Medium | Shadow `store_dedup_match` metric catches this. Root cause: today pinned carousels get `sortOrder = 0,1,2` directly. New path: boost algorithm reserves those positions. If boost algorithm assigns them in slightly different order, dedup walks differently. Fix: ensure config positions exactly match pinned list order. |
| Deal carousel random pinning | Low | `getPinnedDealCarousel()` inserts deal at random position in top 5 pinned slots. New path: deal carousel is `Unboosted` (not in config), ML ranks it. If ML ranks it differently than random pin, shadow diverges. Fix: add deal carousel to config with a position, or accept the divergence as intentional improvement. |
| ICB carousels not in unified config | Low | ICB IDs are currently unioned into `boostByPositionAllowList` at L100. New path: ICB carousels need to be in the unified config JSON. Fix: include ICB carousel IDs in the config during population step. |
| `enablePinnedCarouselsRankScores` DV interaction | Medium | Today this DV controls whether pinned carousels get ML scores. New path: all carousels always get ML scores (no split). If DV is in control (split-before-score), the old path scores only rankable carousels. New path scores everything. Shadow will show score differences for previously-pinned carousels. This is expected and correct — we're moving toward score-everything. |
| Tiger Team `CONTENT_SYSTEM_BOOSTING` override | Low | Old path: overrides allowlist to `dt:` carousels only. New path: reads unified config. Fix: if Tiger Team experiment is active, the unified config should only contain `dt:` prefix entries. Or: keep Tiger Team check in new path as well. |

---

## Timeline

| Week | What |
|---|---|
| **Week 1** | PR 1: Config model + reader + populate JSON from existing configs |
| **Week 2** | PR 2-3: Boost + DHPP changes, unit tests |
| **Week 3** | PR 4: DV setup, dashboard, shadow at 1% |
| **Week 4** | Shadow at 3%, monitor, fix any divergences |
| **Week 5-7** | Serving ramp: 1% → 10% → 100% |
| **Week 8** | Remove old path, delete old configs, delete `PinnedCarouselUtil.kt` |
