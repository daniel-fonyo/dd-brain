# Post-Processing Deep Dive: How Vertical and Horizontal Ranking Actually Work

> Source: `DefaultHomePagePostProcessor.kt` (1092 lines), `HomePagePostProcessorV2.kt`,
> `BaseDedupeConfiguration.kt`, and all supporting utilities.
> This is the job that runs between layout-processing and serialization.

---

## The Big Picture First

Post-processing does four things, in this order:

```
1. VERTICAL RANKING    — decide which carousel goes in which vertical slot
2. TRIM                — drop carousels with too few stores
3. STORE DEDUPE        — remove stores that appear in multiple carousels
4. SORT ORDER FIXUPS   — override positions for NV, PAD, member pricing, gap rules
```

But here's the key nuance: **there are two completely different implementations** that run
depending on a feature flag, and for x-vertical homepage the order is different again.

---

## Which Implementation Runs?

```
reOrder()
    │
    ├─ shouldCleanupSerializer = true        ─────▶  reOrderGlobalEntitiesV2()   ← LIVE path
    │  OR enableNewBodyWorkflow = true
    │
    └─ neither                               ─────▶  reOrderGlobalEntities()      ← DEPRECATED
```

`HomePagePostProcessorV2` is the x-vertical homepage subclass. It overrides `reOrder()`
entirely and runs BEFORE calling the base class for global ranking. More on this below.

---

## Part 1: Regular Homepage (reOrderGlobalEntitiesV2)

### What's the sequence?

```
INPUT: HomePageStoreLayoutOutputElements
  (all carousel types: storeCarousels, itemCarousels, collections, feedPlacements, etc.)
       │
       │  [Parallel async]
       ├─────────────────────────────────────────────────────────────────────┐
       │                                                                     │
       ▼                                                                     ▼
rankAndMergeContent()                               PlacementScoringUtil.scorePlacements()
  (organic carousels through UR + blending)           (NV feed placements scored by MAB)
       │                                                                     │
       └─────────────────────────┬───────────────────────────────────────────┘
                                 │
                                 ▼
              PlacementProcessingUtil.rankAndChoosePlacementsWithOrganic()
                (blend NV placements into the ranked organic carousel list)
                                 │
                                 ▼
                         processContent()
                           ├─ dedupeOrganicAndPlacements()
                           ├─ trimCarousels()
                           ├─ reassignSortOrderBeforeDedupe()
                           ├─ STORE DEDUPE (windowedDedupe)
                           └─ reassignSortOrderAfterDedupe()
                                 │
                                 ▼
             instrumentAndDedupCarousels()   ← safety dedup (carousel-level, not store-level)
                                 │
                                 ▼
              POST-RANKING FIXUPS (in order):
                1. updateSortOrderOfNVCarousel()
                2. updateSortOrderOfTasteOfDashPass()
                3. updateSortOrderOfMemberPricing()
                4. NvCarouselReplacementHandler.applyNvCarouselReplacements()
                5. NonRankableHomepageOrderingUtil.rankWithImmersivesV2()
                   └─ gap rules, color bleed, immersive spacing, final sort re-index
                                 │
                                 ▼
OUTPUT: HomePageStoreLayoutOutputElements
  (every carousel has a final integer sortOrder; sortedPlacements list populated)
```

---

## Part 2: X-Vertical Homepage (HomePagePostProcessorV2)

This is the "cross-vertical" homepage where the page has category sections —
separate sections per vertical (Rx, NV, Grocery, etc.) that each get their own ranked carousels.

The key insight: **vertical sort order matters per section**, so you must rank within
each vertical BEFORE deduping globally. If you dedup first, you might strip a store from
its only vertical section because it appeared in a different vertical section — wrong.

### The sequence is different here:

```
INPUT: layout with categorySections (each section has its own storeCarousels, itemCarousels)
       │
       STEP 1: Rank within EACH category section independently
       ├─ for each CategorySection:
       │    ├─ inject sponsored carousels for this vertical
       │    └─ rankEntities() → full pinning + UR scoring per vertical section
       │
       STEP 2: Rank global carousels (ONLY Favorites, Bookmarks, OpenCarts)
       │    (note: NOT the programmatic carousels — those live in category sections)
       │
       STEP 3: Global dedup across ALL carousels from ALL sections
       │    dedupCarousels(global + every category section's carousels combined)
       │
       STEP 4: Reassign sort orders
       │    ├─ per category section: rebuild with deduped stores, reassign sortOrder
       │    └─ global: same
       │
OUTPUT: layout with ranked categorySections + ranked global carousels, all post-dedup
```

**Why Favorites/Bookmarks/OpenCarts are global but everything else is per-section:**
These carousels span all verticals (your favorite Rx AND your favorite grocery store
both show up in Favorites). They can't be ranked per-vertical.
Programmatic carousels (taste, reorder, etc.) are vertical-specific.

---

## Part 3: Vertical Ranking in Detail (rankAndMergeContent)

This is the core ML ranking step. It decides which carousel goes in which vertical slot.

### The two-step split: Pinned vs Rankable

Every carousel is classified as either **pinned** (fixed position) or **rankable** (ML-scored).

```
splitPinnedAndRankCarousels()
       │
       ▼
PinnedCarouselUtil.getPinnedCarouselRankingRuleList()
       │
       Reads from: carousels/pinned_carousel_ranking_order.json  (runtime JSON)
       Variants:
         - faraway gifts intent    → prepends faraway list
         - nearby rotation v1 DV  → uses nearby rotation list
         - unpin_bookmarks DV     → removes bookmark carousel from pinned list
         - PAD de-merch           → removes TasteOfDashPassCollection
       │
       Returns: ordered list of carousel IDs
       │
       Partition all carousels:
         carousel.id IN rule list → PINNED
         carousel.id NOT in list  → RANKABLE
```

**The experiment-controlled split order:**

```
enablePinnedCarouselsRankScores = false (default)        enablePinnedCarouselsRankScores = true
──────────────────────────────────────────────────────   ────────────────────────────────────────
1. Split into pinned vs rankable                         1. Score EVERYTHING through UR first
2. Send only rankable through UR scoring                 2. Then split into pinned vs rankable
3. Assign pinned: sortOrder = 0, 1, 2...                 3. Assign pinned: sortOrder = 0, 1, 2...
4. Push rankable: sortOrder += pinnedCount               4. Push rankable: sortOrder += pinnedCount
```

The second mode is better (all carousels get UR scores, allowing analysis of what a
pinned carousel WOULD have scored). It's the direction things are moving.

### Deal carousel special case: random pinning

If `shouldUpdateOffersForYouSortOrder` is true, the deal carousel is inserted at a
**random position within the top 5 slots of the pinned list**. This is intentional —
spreading deal carousel position across users to avoid always showing it at slot 0.

```
Random.nextInt(minOf(pinnedList.size + 1, DEAL_CAROUSEL_PIN_RANGE=5))
```

### Pinned carousel sort order assignment

```
rankPinnedCarouselsByRankingRules()

For each ID in pinnedCarouselOrderRuleList (in order):
  ├─ ID = "rt"  (special placeholder)
  │     → find all carousels whose id starts with "rt:"
  │     → assign sortOrder = currIndex++ for each
  │     (this handles realtime carousels that have dynamic UUIDs)
  │
  └─ ID = specific carousel ID
        → find that carousel in the map
        → assign sortOrder = currIndex++

Result: pinned carousels with sortOrder 0, 1, 2...
```

### Rankable carousel sort order assignment

After UR scoring (via `rankContent()` → EntityRankerConfiguration → Sibyl + BlendingUtil + BoostingBundle):

```
mergeContentWithPushedSortOrder()

For every rankable carousel:
  sortOrder = ML-assigned sortOrder + pinnedCarouselCount

Effect: rankable carousels can never rank above pinned carousels.
Pinned always occupy 0..N-1. Rankable starts at N.
```

---

## Part 4: Trim Carousels

Before deduplication, carousels with too few stores are dropped entirely.

```
trimCarousels()
       │
       Per carousel type:
       ├─ TALL_LOGO_MERCH (MX logo carousel):
       │     keep if: storeCount >= minStoresForMxLogoCarousel AND NVMxLogo DV enabled
       │
       ├─ LIST_COMPONENT / STORE_LIST_UNIVERSAL_CAROUSEL:
       │     keep if: storeCount >= 2 (MIN_STORES_FOR_GROCERY_LIST_FORMAT)
       │             AND passes minStores check (below)
       │
       └─ everything else:
             keep if: storeCount >= minStores
                      OR primaryVerticalId is in vertical exclusion list (some verticals exempt)
                      OR carouselId is in storeCarouselBlockList (always keep)
             AND NOT in cappedCarouselIds (CarouselCapping)

minStores default: from DiscoveryExperimentManager.findMinStoresExpParams()
                   configurable per pageType via DV
```

**CarouselCapping** is a separate concept: some carousels are "capped" by promotion/campaign
config and should be included regardless of store count (or excluded regardless of store count).
The trim step respects these.

---

## Part 5: Store Deduplication

This is the most complex piece. The goal: if a store appears in multiple carousels,
it should only appear in the highest-priority carousel, not all of them.

### Two types of dedupe — don't confuse them

```
TYPE 1: windowedDedupe (BaseDedupeConfiguration)
        Removes STORES that appear in multiple carousels.
        "If store X is in Favorites AND in Taste carousel, keep it only in Favorites"
        This is the main dedupe. It touches store lists within carousels.

TYPE 2: instrumentAndDedupCarousels (DefaultHomePagePostProcessor)
        Removes DUPLICATE CAROUSELS with the same carousel ID.
        "If somehow the same carousel appears twice in the list, keep only the first"
        This is a safety net / bug guard. It should never fire in practice.
```

### How windowedDedupe works (TYPE 1)

The algorithm uses a sliding window — within a horizontal window size (slots) and
vertical window size (carousels), once a store is seen, it's removed from subsequent appearances.

```
windowedDedupe(sortedCarousels, itemCarousels)
       │
       STEP 1: Convert all carousel types to DedupeCarousel
               StoreCarousel  → DedupeCarouselType.STORE_CAROUSEL
               ItemCarousel   → DedupeCarouselType.ITEM_CAROUSEL
               intermixedStores → wrapped as single-store StoreCarousels
       │
       STEP 2: Sort DedupeCarousels by sortOrder
               (this is why ranking MUST happen before dedupe —
               the dedup algorithm is position-sensitive)
       │
       STEP 3: Walk carousels in vertical order (top to bottom)
               Maintain a seen-stores set
               For each carousel:
                 For each store in the carousel:
                   IF store.id already seen in recent window:
                     REMOVE store from this carousel
                   ELSE:
                     ADD store.id to seen-stores
       │
       STEP 4: Remove carousels that became empty after dedupe
```

**Window configuration** (from runtime JSON `dedupe_window_config.json`):

```
horizontalWindowSize   — how many stores horizontally to look back within a carousel
verticalWindowSize     — how many carousels vertically to look back
allowRepeatedDefault   — if true, a store CAN appear again after the window expires
combinedVerticalDepth  — for combined (store + item) dedup, how deep to look
```

Default: both windows = `Int.MAX_VALUE` (see all previously seen stores across all carousels).
`allowRepeatedDefault = false` (once removed, stays removed).

This means by default: if you appear in Favorites, you will not appear in ANY other carousel.
Favorites wins because it has a lower sortOrder and is seen first.

### X-vertical dedupe special case: `shouldSortCombinedCarousels = false`

```
V2 (x-vertical):
  dedupCarousels(storeCarousels = global + every category section's carousels)
  shouldSortCombinedCarousels = false  ← KEY

V1 (regular):
  dedupe(shouldSortCombinedCarousels = true)
```

When `shouldSortCombinedCarousels = false`, the carousels are NOT re-sorted by sortOrder
before dedup. Why? Because each category section has already been ranked internally,
and the inter-section order is meaningful. Mixing and re-sorting by a global sortOrder
would break the intra-section relative ordering that was carefully established in Step 1.

This is the "carousel ranking matters before dedupe" comment from the pipeline code.

---

## Part 6: Sort Order Fixups (Post-Ranking Overrides)

After all ranking and dedupe, these run in strict order. **Each can override everything before it.**

### Fixup 1: NV carousel post-checkout pinning

```
updateSortOrderOfNVCarousel()
       │
       └─ isPostCheckoutNVContentBoosting = true
          (user completed an order in the last 60 minutes)
               │
               ├─ shouldEnablePostCheckoutNVGroceryOnlyBoosting = true
               │     Only grocery vertical (verticalId = 3) carousels affected
               │
               └─ else: all carousels in postCheckoutNVContentBoostingCarouselIds list
               │
               Top 3 NV carousels by current sortOrder → pinned to positions 0, 1, 2
```

**Why this exists:** After a user orders, show NV (non-restaurant) content prominently
to encourage cross-vertical discovery while the user is engaged. Time-limited.

### Fixup 2: Taste of DashPass (PAD)

```
updateSortOrderOfTasteOfDashPass()
       │
       └─ PAD_CAROUSEL_SPOTLIGHT_V2 DV = treatment1
               TasteOfDashPassCollection → sortOrder = 3
```

Hardcoded to position 3. No ML involved.

### Fixup 3: Member Pricing

```
updateSortOrderOfMemberPricing()
       │
       └─ isMemberPricingV2PriorityCarouselsEnabled = true
               First carousel matching priority list → sortOrder = 0
               (pushes everything else down by 1)
```

Hardcoded to position 0. Wins over everything including NV post-checkout pinning
because it runs after fixup 1. The last fixup to run on a slot wins.

### Fixup 4: NV Carousel Replacement

```
NvCarouselReplacementHandler.applyNvCarouselReplacements()
       │
       └─ Experiment-gated NV carousel replacement logic
          Certain NV carousels swapped for alternative versions based on context
          (e.g., replacing a generic NV carousel with a personalized one)
```

### Fixup 5: Final Ordering — Gap Rules, Color Bleed, Immersive Spacing

This is the last and most complex fixup. It merges ALL content types into a single flat
sorted list and then applies spacing rules.

```
NonRankableHomepageOrderingUtil.rankWithImmersivesV2()
       │
       adjustSortOrdersWithTieBreaking()
         │
         ├─ toSortedListWithTieBreaking()
         │     Merges ALL content types into one flat list:
         │     [storeCarousels, itemCarousels, collections, feedPlacements,
         │      dineOutCarousel, reelsCarousel, bannerCarousel, spotlightBanner,
         │      serviceFeeBanner, smartSuggestionCollection, dashPartyCarousel,
         │      mapCarousels, doubleDashReminder, avProductCarousel]
         │     Sorted by sortOrder, ties broken by type priority
         │
         ├─ reorderPlacementsForColorBleed()
         │     NV placement carousels regrouped to avoid adjacent same-color placements
         │     (visual quality rule — prevents two blue NV banners next to each other)
         │
         └─ GapRulesUtil.reorderForImmersiveContentSpacing()
               Immersive content = visually large items (ads, banners, NV placements)
               Rules:
               ├─ Max 3 immersive items total (GAP_RULES_LIMIT, DV-overridable)
               ├─ Each immersive item has getRequiredGap() → min organic items between immersives
               ├─ Each immersive item has getMaxVerticalPlacement() → won't appear above this slot
               ├─ If gap constraint not satisfied → defer item to ArrayDeque, try later
               ├─ If item still can't fit after deferred → append at end
               ├─ Safety: if reordering drops ALL immersive content → revert to original
               └─ ServiceFeeBanner, SmartSuggestionCollection: exempt from gap counting
               │
               Re-index: final sortOrder = 0, 1, 2, 3... (contiguous integers)
               ← This is the FINAL sort order that the serializer uses
```

**`sortedPlacements`** — the output of this step is stored as `sortedContents: List<SortablePlacement>`.
This is the authoritative ordered list. The serializer uses this to determine the page layout.
This is also what the Iguazu event should use for `facet_vertical_position` (not `carousel.sortOrder`).

---

## Part 7: What storeList sortOrder is

The store list (infinite scroll fallback) always goes last:

```
storeListSortOrder = max(
  all storeCarousel sortOrders,
  all itemCarousel sortOrders,
  all collection sortOrders,
  dealCarousel sortOrder,
  mapCarousel sortOrders,
  intermixedStore sortOrders
) + 1
```

In `reOrderGlobalEntitiesV2` the final storeList sortOrder = `sortedContents.size`
(i.e., the next slot after the last ranked item). This ensures store list always appears
below all carousels.

---

## Edge Cases and Nuances Summary

| Situation | What happens |
|---|---|
| Carousel appears in pinned list BUT store count is below minStores | trimCarousels() drops it BEFORE rankPinnedCarouselsByRankingRules(), so it never gets pinned |
| Deal carousel with shouldUpdateOffersForYouSortOrder | Randomly inserted within top 5 pinned slots. Position varies per request |
| Taste carousel ID format: `"taste:pizza"` (contains `:`) | `getCarouselRankingKey()` splits on `:` and uses only the first part for pinned list matching |
| `GeneratedRecommendationProduct` carousel IDs (cxgen) | Start with a known prefix; NOT split on `:` despite containing it |
| `"rt"` in pinned list | Special placeholder — expands to all carousels whose ID starts with `"rt:"` (realtime carousels) |
| Store in Favorites AND Taste carousel | Favorites wins (lower sortOrder → seen first in dedup walk) |
| `enablePinnedCarouselsRankScores = true` | ALL carousels scored by UR before pinning. Pinned carousels get UR scores for analytics even though position is fixed |
| X-vertical homepage has `categorySections = null` | V2 processor treats it as empty map → falls through to ranking global carousels only. Behaves same as regular homepage in that case |
| `ENABLE_DEDUPE_ITEM_CAROUSELS` DV off | Item carousels are NOT deduped across category sections, even if a store appears in item carousels in multiple verticals |
| Post-ranking fixups run after dedup | A carousel deduped down to 0 stores CAN still get a sortOrder assigned by a fixup. But when serialized, an empty carousel should be filtered — this is a known fragility |
| NV post-checkout pinning (fixup 1) AND member pricing (fixup 3) both active | Member pricing always beats NV post-checkout because it runs last. Position 0 always goes to member pricing if that DV is active |
| `sortedPlacements` empty | Downstream code (serializer) treats this as V1 path. If V1 is deprecated, the serializer falls back gracefully |

---

## What This Means for UBP

The post-processing pipeline is the most entangled part of the codebase. Four things
that make it hard to experiment on:

**1. Two separate ranking passes with different scopes**
In V2 (x-vertical), ranking happens per-category-section THEN global. Adding a new
ranking signal requires touching both.

**2. Fixups are outside the ranked pipeline**
Steps 1-5 in the post-ranking fixups directly set `sortOrder` on domain objects after
all the ML ranking is done. They're not in the DAG. They're not traced.
When a carousel ends up at position 3, you don't know if that was the UR model or
the PAD fixup or the member pricing fixup.

**3. Dedup is position-sensitive and order-dependent**
You cannot change the order in which carousels are ranked without also changing which
stores survive dedup. Ranking and dedup are coupled.

**4. `sortedPlacements` is the ground truth, but `sortOrder` on domain objects is the proxy**
The serializer uses `sortedPlacements` (the output of `rankWithImmersivesV2`).
The Iguazu events use `sortOrder` on individual carousels. These can diverge when
gap rules reorder things. This is the vertical position discrepancy problem.

**UBP's job**: make steps 1-5 in the post-ranking fixups into declared pipeline steps
with automatic tracing. This makes every position decision observable.
