# Carousel Positioning: Exhaustive Config & Decision Audit

**Date:** 2026-03-30
**Purpose:** Complete map of every config, DV, and code path that determines carousel position on the homepage. This is the spec for a future unified positioning config.
**Scope:** Homepage pipeline only (not retail, VLP, or shopping tabs).

---

## How to Read This Document

A carousel's final position is the result of a **pipeline of overwrites**. The carousel enters with an upstream `sortOrder`, passes through ranking (which may reassign it), then passes through post-ranking fixups (which may overwrite it again). The **last write wins**.

This document traces every write site in pipeline order.

---

## Phase 0: Upstream — CMS/Catalog Sets Initial Position

Before feed-service touches a carousel, it arrives with two fields set by upstream systems (CMS, catalog, taxonomy tags):

| Field | Type | Set By | Meaning |
|---|---|---|---|
| `sortOrder` | `Int?` | CMS catalog config | Desired position (0-indexed) |
| `enforceManualSortOrder` | `Boolean` | CMS catalog config | If true, this carousel demands a fixed position |

**Where `enforceManualSortOrder` is set to `true` in feed-service code:**

| Source | File | What |
|---|---|---|
| Discovery tile collections | `DiscoveryTileCollectionsUtil.kt` L18, L31 | Always `true` for tile collections from ProgrammaticProduct |
| NV item carousel tags | `NVItemAttributeUtil.kt` L353, L405 | `true` when `tag.isSortOrderPinned == true` (data-driven from taxonomy) |
| ICB service | `ImpressionCappedBoostingService.kt` L146 | Set `true` when applying ICB boost (forces position) |
| Operator collections (retail) | `ItemCarouselOperatorCollectionContentFetching.kt` L371 | Set `true` for operator-ordered retail carousels |

**Key insight:** `enforceManualSortOrder` is the CMS-level "pin" that the ranking pipeline reads but never creates (except ICB). It's invisible to all our runtime allowlists.

---

## Phase 1: Pinned/Rankable Split

**File:** `DefaultHomePagePostProcessor.kt` L766-814
**Method:** `splitPinnedAndRankCarousels()`

Every carousel is classified as **pinned** (fixed position, ML cannot touch it) or **rankable** (ML-scored). This happens BEFORE any ML ranking.

### Config: Pinned Carousel Ordering

| Config File | When Used | DV Gate |
|---|---|---|
| `carousels/pinned_carousel_ranking_order.json` | Default homepage | None (always) |
| `carousels/pinned_carousel_ranking_order_faraway.json` | Faraway gifts intent | `GiftsExperimentManager.TREATMENT_PINNED` |
| `carousels/pinned_carousel_ranking_order_nearby_rotation_v1.json` | Nearby rotation treatment | `PADUtil.isAnyNearbyRotationV1Treatment()` AND user is NOT DashPass |
| `carousels/pinned_carousel_me_tab_ranking_order.json` | ME tab | Page type = ME tab |
| `carousels/pinned_carousel_token_page_ranking_order.json` | Token page | Page type = token page |

**Read by:** `PinnedCarouselUtil.getPinnedCarouselRankingRuleList()` (L121-151)

### DV Filters on the Pinned List

| DV | Effect | File:Line |
|---|---|---|
| `discovery_p13n_unpin_bookmarks` | Removes specific carousel IDs from pinned list | `PinnedCarouselUtil.kt` L104-116, L177 |
| `PADUtil.shouldUnpinTasteOfDPCollection()` | Removes `TasteOfDashPassCollection` from pinned list | `PinnedCarouselUtil.kt` L140 |

### Special Cases in Pinned List

| Case | Logic | File:Line |
|---|---|---|
| `"rt"` in pinned list | Expands to ALL carousels whose ID starts with `"rt:"` (realtime carousels) | `PinnedCarouselUtil.kt` L63-72 |
| Taste carousel IDs | Split on separator (`:`) — only prefix used for matching | `PinnedCarouselUtil.kt` L74-82 |
| Deal carousel random pinning | If `shouldUpdateOffersForYouSortOrder()`, deal carousel inserted at random position within top 5 pinned slots | `DefaultHomePagePostProcessor.kt` L824-863 |

### sortOrder Assignment

**Method:** `PinnedCarouselUtil.rankPinnedCarouselsByRankingRules()` (L37-99)

Pinned carousels get `sortOrder = 0, 1, 2...` in the order they appear in the rule list. Rankable carousels get pushed: `sortOrder += pinnedCount` (via `mergeContentWithPushedSortOrder()`).

**Result:** Pinned carousels always occupy positions 0..N-1. Rankable carousels start at position N.

---

## Phase 2: ML Ranking (EntityRankerConfiguration)

Rankable carousels (those NOT in the pinned list) enter the ranking pipeline. This is where `Boosting.kt` runs.

### 2a. Boost Allowlist — Who Can Be Position-Boosted

**Config:** `boost_by_position_carousel_id_allow_list.json`
**Read by:** `PersonalizationRuntimeUtil.boostByPositionCarouselIdAllowList()` (L236-242)
**Used at:** `Boosting.kt` L90-102

| Condition | Allowlist Source | File:Line |
|---|---|---|
| `TigerTeamFeature.CONTENT_SYSTEM_BOOSTING` enabled | Override: only carousel IDs starting with `"dt:"` (dynamic taste) | `Boosting.kt` L90-96 |
| Normal (no Tiger Team) | `PersonalizationRuntimeUtil.boostByPositionCarouselIdAllowList()` UNION `impressionCappedBoostingCarouselIds` | `Boosting.kt` L99-101 |
| Allowlist contains `"ALL"` | Every carousel is eligible | `Boosting.kt` L336-342 |

### 2b. Boost Hint Generation — Which Carousels Request a Position

Each `BoostingCandidate` type has its own `getBoostingHint()` logic:

| Type | Condition for `Boosted(position)` | Condition for `Unboosted` | File:Line |
|---|---|---|---|
| **StoreCarousel** | `sortOrder != null` AND `enforceManualSortOrder == true` | Otherwise | L494-508 |
| **ItemCarousel** | `sortOrder != null` AND `enforceManualSortOrder == true` | Otherwise | L602-616 |
| **MapCarousel** | `sortOrder != null` AND `enforceManualSortOrder == true` | Otherwise | L682-701 |
| **StoreCollection** | `enforceManualSortOrder == true` | Otherwise | L530-544 |
| **CollectionV2** | `enforceManualSortOrder == true` | Otherwise | L566-580 |
| **ItemCollection** | `enforceManualSortOrder == true` | Otherwise | L638-651 |
| **ReelsCarousel** | `enforceManualSortOrder == true` | Otherwise | L446-460 |
| **DealCarousel** | Never | Always `Unboosted` | L436 |
| **StoreEntity** | Never | Always `Unboosted` | L661 |

**Key inconsistency:** StoreCarousel, ItemCarousel, MapCarousel check `sortOrder != null` in addition to `enforceManualSortOrder`. Collections only check `enforceManualSortOrder`. This means a collection with `enforceManualSortOrder=true` but `sortOrder=null` would get `Boosted(position=null)` — likely a latent bug.

### 2c. NV Unpin — Stripping Boost from Non-Restaurant Carousels

**Gate DVs:**
- `P13nExperimentManager.enableHomepageVerticalBlending(experimentMap)` — must be true
- `pageType.isHomepage()` — must be true

**Logic:** `Boosting.kt` L131-160

```
For each carousel in toBoostPairs:
  IF isEligibleForUnpin(candidate) == true:
    IF AFM exemption DV is NOT control AND isExemptAfmCarousel(candidate):
      KEEP (AFM exemption)
    ELSE:
      REMOVE from toBoostPairs → move to unBoostPairs (ML-ranked freely)
```

**`isEligibleForUnpin()`** (L693-714): Returns `true` if carousel's first store's `primaryVerticalIds` is non-empty AND does NOT contain `RESTAURANT_VERTICAL_ID` (1). Only checks `StoreCarousel` and `ItemCarousel` — all other types return `null` → never unpinned.

**AFM Exemption** (L741-765):
- DV: `ENABLE_AFM_UNPIN_EXEMPTION` — must NOT be control
- Checks: carousel has `AFFORDABLE_MEALS_VERTICAL_ID` AND active meal-box campaign
- Uses same badge logic as `StoreOfferBadgeServiceV2`

**THE BUG:** The `boostByPositionAllowList` is NOT checked during unpin. A carousel can be in the allowlist and still get unpinned. See [AFM analysis](./afm-unpin-bug-and-boost-cleanup.md).

### 2d. Deal Carousel Score Multiplier

**DV:** `DiscoveryExperimentUtil.getDealCarouselScoreMultiplier(experimentMap)` (Boosting.kt L176)
**Effect:** Multiplies unboosted deal carousel's prediction score before sorting. Lower multiplier = lower position.

### 2e. Impression Capped Boosting (ICB)

**Service:** `ImpressionCappedBoostingService.kt`
**Feature gate:** `IMPRESSION_CAPPED_BOOSTING_FEATURE_FLAG` (DiscoveryExperimentManager)

**Config source:** `ImpressionCappedBoostingConfig` DV experiment (`530621f9-095a-4588-bf4c-1572383c4fbc`)

| Config Field | What |
|---|---|
| `experiments[].entityTitles` | Set of carousel TITLES (not IDs!) eligible for ICB |
| `experiments[].boostingRule.boostedPosition` | Position to pin to (default: 5) |
| `experiments[].boostingRule.impressionsCap` | Max impressions before capping (default: 5) |
| `experiments[].boostingRule.preCapRule` | `BOOST` or `DO_NOT_BOOST` |
| `experiments[].boostingRule.postCapRule` | `SHOW_WITH_DEFAULT_ORDERING` or `DO_NOT_SHOW` |

**ICB applies boost by:** Setting `sortOrder = boostedPosition` AND `enforceManualSortOrder = true` on the carousel. This feeds into the `getBoostingHint()` logic above.

**LCM M2 special logic** (L176-246):
- Gate: `DiscoveryExperimentManager.shouldEnableLcmM2UseCaseCarousel()` returns treatment variant
- Carousels with `groupedItemCarousels` scored by weighted item score (0.6/0.3/0.1 for first 3 items)
- Positions from runtime config: `DiscoveryRuntimeUtil.getLcmCarouselsVerticalPositionsByTreatmentArm()`
- Default: `TREATMENT_1 → [4, 7, 10, 13]`, `TREATMENT_2 → [3, 6, 9, 12]`
- Excluded: `LCM_M2_MVP_TEST_CAROUSEL_Id = "627c8535-7d71-477f-908d-66bcb8365e94"`

### 2f. Carousel Capping — Who Gets Hidden

**File:** `CarouselCapping.kt`

**Master gate:** `ENABLE_CAROUSEL_CAPPING` DV → `PersonalizationExperimentManager.getCarouselCappingParams()`

| Config | Source | Default | What |
|---|---|---|---|
| `shouldEnableCarouselCapping` | DV treatment check | false | Master on/off |
| `carouselCappingThreshold` | DV or UR fallback | 0.2 | % of stores above which carousel is kept |
| `carouselBlockList` | `carousel_capping_blocklist.json` | empty | Carousel IDs NEVER capped |
| `redisplayProbability` | `carousel_capping_redisplay_probability.float` | 0.125 | Chance a capped carousel resurfaces |
| `withNvConstraint` | DV | true | Separate Rx/NV thresholds |
| `shouldCapResurfacing` | DV | false | Limit resurfacing to 1 per category |

**Exemptions from capping:**

| Exemption | Condition | File:Line |
|---|---|---|
| Blocklist | `carousel.id IN carouselBlockList` (handles taste suffix via `:` split) | `CarouselCapping.kt` L338-346 |
| ICB carousels | `IMPRESSION_CAPPED_BOOSTING_FEATURE_FLAG` enabled AND carousel title in ICB eligible titles | `CarouselCapping.kt` L312-327 |
| Tiger Team | `TigerTeamFeature.DISABLE_THE_CAROUSEL_CAPPING` enabled AND carousel prefix != `"dt:"` | `CarouselCapping.kt` L332-341 |

**V1 algorithm (default):** With NV constraint = partition into Rx (contains restaurant vertical) and non-Rx, apply threshold separately. Without = apply uniformly.

### 2g. sortOrder Assignment in Ranking

**File:** `Ranking.kt` L126-134 (`RankingBundle.ranked()`)

The boost algorithm produces a final ordered list. `Ranking.kt` assigns `sortOrder` based on position in that list via a `Positioner` that reserves slots for pinned entities and fills the rest sequentially.

---

## Phase 3: Dedup and Trim

**File:** `DefaultHomePagePostProcessor.kt` → `processContent()`

### Trim (before dedup)
Carousels with too few stores are dropped. Controlled by:
- `DiscoveryExperimentManager.findMinStoresExpParams()` — minimum store count per page type
- `storeCarouselBlockList` — carousels always kept regardless of count
- Vertical exclusion list — some verticals exempt from min-store requirement
- `cappedCarouselIds` — carousels capped by `CarouselCapping` are dropped

### Windowed Dedup
Position-sensitive: higher-ranked carousels win. Dedup walks carousels in `sortOrder` order — a carousel at position 2 keeps its stores; the same store in a carousel at position 8 gets removed.

**Implication for unified config:** Changing a carousel's position changes which stores survive dedup. Moving a carousel from position 2 to position 10 could cause it to lose stores to earlier carousels.

---

## Phase 4: Post-Ranking Fixups (Last Write Wins)

These run IN THIS ORDER after dedup. Each can overwrite all previous sortOrder assignments. **The last fixup to write a position wins.**

### Fixup 1: NV Carousel Post-Checkout Pinning

**File:** `NonRankableHomepageOrderingUtil.kt` L269-291
**Gate:** `isPostCheckoutNVContentBoosting` (user completed an order in last 60 minutes)

| Sub-gate | Effect |
|---|---|
| `shouldEnablePostCheckoutNVGroceryOnlyBoosting = true` | Only grocery vertical (verticalId=3) carousels affected |
| Otherwise | All carousels in `postCheckoutNVContentBoostingCarouselIds` list |

**Effect:** Top 3 matching NV carousels by current sortOrder → pinned to positions 0, 1, 2.

### Fixup 2: Taste of DashPass (PAD) Pinning

**File:** `NonRankableHomepageOrderingUtil.kt` L299-318
**Gate:** `PAD_CAROUSEL_SPOTLIGHT_V2` DV

| Treatment | Effect |
|---|---|
| `treatment1` | `TasteOfDashPassCollection` → `sortOrder = 3` |
| `control` or `treatment2` | No change (stays at position 0 from pinned list) |

### Fixup 3: Member Pricing Priority

**File:** `NonRankableHomepageOrderingUtil.kt` L229-248
**Gate:** `MemberPricingUtil.isMemberPricingV2PriorityCarouselsEnabled(experimentMap)`
**Config:** `DiscoveryRuntimeUtil.getPriorityCarouselsWithDPMemberPricing()` — list of carousel IDs

**Effect:** First matching carousel → `sortOrder = 0`. Wins over everything because it runs after fixups 1 and 2.

### Fixup 4: NV Carousel Replacement

**File:** `NvCarouselReplacementHandler.kt`
**Gate:** `discoveryExperimentManager.shouldEnableNvCarouselReplacements()`
**Config:** `DiscoveryRuntimeUtil.getNvCarouselReplacementConfig()` — runtime JSON rules

| Strategy | DV | Effect |
|---|---|---|
| BIA replacement | `getNvCarouselBiaReplacementVariant()` | Replaces NV carousel with BIA variant |
| Alcohol tiles | `getAlcoholTilesOnHomepageVariant()` | Replaces NV carousel with alcohol tiles |
| Drop | Config rule | Removes NV carousel entirely |

### Fixup 5: Spotlight Banner Position

**File:** `NonRankableHomepageOrderingUtil.kt` L141-179
**Gate:** `shouldEnableImmersiveBanner()`

| Condition | Position |
|---|---|
| Campaign is in allowlist | `sortOrder = 0` |
| Otherwise | `sortOrder = DiscoveryRuntimeUtil.getImmersiveBannerPosition()` |

### Fixup 6: Color Bleed Reordering

**File:** `NonRankableHomepageOrderingUtil.kt` L97-104
**Effect:** Reorders NV feed placements to avoid adjacent same-color placements. Can shift carousel positions.

### Fixup 7: Gap Rules + Immersive Spacing

**File:** `GapRulesUtil.reorderForImmersiveContentSpacing()`
**Effect:** Inserts spacing between immersive content (ads, banners, NV placements). Rules:
- Max 3 immersive items total (`GAP_RULES_LIMIT`, DV-overridable)
- Each immersive item has `getRequiredGap()` — min organic items between immersives
- Each has `getMaxVerticalPlacement()` — won't appear above this slot
- If gap constraint not satisfied → defer to end or drop

### FINAL: Re-indexing

**File:** `NonRankableHomepageOrderingUtil.kt` L113-115

```kotlin
immersiveReorderList.mapIndexed { index, placement ->
    placement.assignSortOrder(index)
}
```

**This is the LAST sortOrder write.** Everything above is overwritten. The final physical order (after all fixups, gap rules, and immersive spacing) is mapped to sequential `0, 1, 2, 3...`. This is what the serializer uses. This is what the user sees.

---

## Complete Decision Point Inventory

### Runtime JSON Config Files (7)

| # | File | Read By | Controls |
|---|---|---|---|
| 1 | `carousels/pinned_carousel_ranking_order.json` | `PinnedCarouselUtil` | Base pinned carousel order |
| 2 | `carousels/pinned_carousel_ranking_order_faraway.json` | `PinnedCarouselUtil` | Faraway intent pinned order |
| 3 | `carousels/pinned_carousel_ranking_order_nearby_rotation_v1.json` | `PinnedCarouselUtil` | Nearby rotation pinned order |
| 4 | `boost_by_position_carousel_id_allow_list.json` | `PersonalizationRuntimeUtil` | Boost eligibility |
| 5 | `carousel_capping_blocklist.json` | `PersonalizationRuntimeUtil` | Capping exemption |
| 6 | `carousel_capping_redisplay_probability.float` | `PersonalizationRuntimeUtil` | Resurfacing probability |
| 7 | `lcm_carousels_vertical_positions_by_treatment_arm.json` | `DiscoveryRuntimeUtil` | LCM M2 positions per treatment |

### DV Experiment Gates (18)

| # | DV / Experiment | Controls | Used At |
|---|---|---|---|
| 1 | `discovery_p13n_unpin_bookmarks` | Remove carousel IDs from pinned list | `PinnedCarouselUtil` L177 |
| 2 | `GiftsExperimentManager.TREATMENT_PINNED` | Use faraway pinned list | `PinnedCarouselUtil` L123 |
| 3 | `PADUtil.shouldUnpinTasteOfDPCollection()` | Unpin Taste of DashPass from pinned list | `PinnedCarouselUtil` L140 |
| 4 | `PADUtil.isAnyNearbyRotationV1Treatment()` | Use nearby rotation pinned list | `PinnedCarouselUtil` L128 |
| 5 | `shouldUpdateOffersForYouSortOrder()` | Deal carousel random pin within top 5 | `DHPP` L824 |
| 6 | `enableHomepageVerticalBlending` | Gate NV unpin logic | `Boosting.kt` L135 |
| 7 | `ENABLE_AFM_UNPIN_EXEMPTION` | Exempt AFM from NV unpin | `Boosting.kt` L745 |
| 8 | `TigerTeamFeature.CONTENT_SYSTEM_BOOSTING` | Override boost allowlist to `dt:` only | `Boosting.kt` L90 |
| 9 | `getDealCarouselScoreMultiplier()` | Multiply deal carousel score | `Boosting.kt` L176 |
| 10 | `IMPRESSION_CAPPED_BOOSTING_FEATURE_FLAG` | Gate ICB system | `ImpressionCappedBoostingService` |
| 11 | `ENABLE_LCM_M2_USE_CASE_CAROUSELS` | Gate LCM M2 special positioning | `ImpressionCappedBoostingService` L223 |
| 12 | `ENABLE_CAROUSEL_CAPPING` | Gate capping + set threshold | `PersonalizationExperimentManager` L243 |
| 13 | `UNIVERSAL_RANKER_V5/FW/FW_DELAY` | Override capping threshold for UR holdouts | `PersonalizationExperimentManager` L252-261 |
| 14 | `TigerTeamFeature.DISABLE_THE_CAROUSEL_CAPPING` | Exempt non-DT carousels from capping | `CarouselCapping.kt` L334 |
| 15 | `isPostCheckoutNVContentBoosting` | NV post-checkout pinning to 0,1,2 | `NonRankableHomepageOrderingUtil` L269 |
| 16 | `PAD_CAROUSEL_SPOTLIGHT_V2` | PAD → position 3 on treatment1 | `NonRankableHomepageOrderingUtil` L300 |
| 17 | `isMemberPricingV2PriorityCarouselsEnabled()` | Member pricing → position 0 | `NonRankableHomepageOrderingUtil` L230 |
| 18 | `shouldEnableNvCarouselReplacements()` | NV carousel swap/drop | `NvCarouselReplacementHandler` L36 |

### Hardcoded Values (6)

| # | Value | What | File:Line |
|---|---|---|---|
| 1 | `RESTAURANT_VERTICAL_ID = 1` | NV unpin check | `Boosting.kt` L725 |
| 2 | `AFFORDABLE_MEALS_VERTICAL_ID = 100322` | AFM exemption check | `Boosting.kt` L760 |
| 3 | `DEAL_CAROUSEL_PIN_RANGE = 5` | Random pin range for deal carousel | `DHPP` L94 |
| 4 | PAD position = `3` | Hardcoded PAD position on treatment1 | `NonRankableHomepageOrderingUtil` L311 |
| 5 | Member pricing position = `0` | Hardcoded member pricing position | `NonRankableHomepageOrderingUtil` L258 |
| 6 | NV post-checkout positions = `0, 1, 2` | Hardcoded top 3 NV positions | `NonRankableHomepageOrderingUtil` L276-289 |

### Upstream (CMS/Catalog) Inputs (2)

| # | Field | Set By | Effect on Pipeline |
|---|---|---|---|
| 1 | `enforceManualSortOrder` | CMS catalog, taxonomy tags, ICB service | `getBoostingHint()` returns `Boosted(position)` → enters boost algorithm |
| 2 | `sortOrder` | CMS catalog, taxonomy tags, ICB service | Used as the target position when `enforceManualSortOrder = true` |

---

## sortOrder Write Sites (Chronological)

Every place `sortOrder` is assigned, in pipeline execution order:

| # | When | What | Value | File:Line |
|---|---|---|---|---|
| 1 | Data fetching | Operator collection ordering | `operatorOverrideOrder` or intention-driven | `ItemCarouselOperatorCollectionContentFetching.kt` L357 |
| 2 | ICB application | ICB boost applied | `boostingRule.boostedPosition` | `ImpressionCappedBoostingService.kt` L146 |
| 3 | Pinned split | Pinned carousels ordered | `0, 1, 2...` per rule list | `PinnedCarouselUtil.kt` L37-99 |
| 4 | Pinned split | Rankable carousels pushed | `+= pinnedCount` | `DHPP.mergeContentWithPushedSortOrder()` |
| 5 | LCM M2 intra-ranking | LCM treatment positions | `LCM_TREATMENT_X_SORT_ORDERS[index]` | `EntityRankerConfiguration.kt` L723 |
| 6 | Boost algorithm | 4-pass position assignment | Positioner result (reserved + sequential) | `Ranking.kt` L126-134 |
| 7 | Post-dedup | NV post-checkout pinning | `0, 1, 2` for top 3 NV | `NonRankableHomepageOrderingUtil.kt` L276-289 |
| 8 | Post-dedup | PAD spotlight | `3` (treatment1) | `NonRankableHomepageOrderingUtil.kt` L311 |
| 9 | Post-dedup | Member pricing | `0` | `NonRankableHomepageOrderingUtil.kt` L258 |
| 10 | Post-dedup | Spotlight banner | `0` or `getImmersiveBannerPosition()` | `NonRankableHomepageOrderingUtil.kt` L176 |
| 11 | Post-dedup | Color bleed reorder | Shifted by reorder | `NonRankableHomepageOrderingUtil.kt` L97-104 |
| 12 | Post-dedup | Gap rules + immersive spacing | Shifted by gap insertion | `GapRulesUtil.reorderForImmersiveContentSpacing()` |
| **13** | **FINAL** | **Re-indexing** | **`0, 1, 2, 3...` (mapIndexed)** | **`NonRankableHomepageOrderingUtil.kt` L113-115** |

**Write #13 is the ground truth.** Everything above it is an intermediate. The serializer and Iguazu events should read from the output of write #13.

---

## What This Means for a Unified Config

### What can be unified (same conceptual question: "should this carousel be at a fixed position?")
- Pinned carousel list (#1-4 above)
- Boost allowlist (#4 above)
- ICB config (#2 above, but needs ID migration from title matching)
- Post-ranking fixups (#7-10 above)

### What cannot be unified (different conceptual questions)
- **Capping** — "should this carousel be hidden?" is a separate question from "where should it be?"
- **Gap rules** — layout/visual concern, not positioning intent
- **Color bleed** — visual concern
- **Dedup** — position-sensitive side effect, not intent

### Prerequisites for unification
1. **ICB must migrate from title matching to ID matching** — currently matches by `entityTitles` (e.g., "Grocery"), not carousel ID
2. **Post-ranking fixups must become data-driven** — currently hardcoded positions in code (PAD=3, member pricing=0, NV post-checkout=0,1,2)
3. **`enforceManualSortOrder` must be documented per carousel** — CMS-set pins are invisible to our configs; need inventory of which carousels have this flag set upstream
4. **Stale entries must be cleaned** — allowlist has legacy NV entries; pinned list may have dead carousel IDs

---

## Key Files Quick Reference

| File | Lines | What |
|---|---|---|
| `Boosting.kt` | 765 | Boost algorithm, allowlist, NV unpin, AFM exemption, all candidate hint logic |
| `PinnedCarouselUtil.kt` | 183 | Pinned list construction, JSON variant selection, DV filtering |
| `CarouselCapping.kt` | 419 | V1/V2 capping, blocklist, ICB exemption, Tiger Team override |
| `ImpressionCappedBoostingService.kt` | 247 | ICB system, M2 scoring, UPSS integration |
| `PersonalizationRuntimeUtil.kt` | ~400 | Boost allowlist reader, capping blocklist reader, capping params |
| `DiscoveryRuntimeUtil.kt` | ~3000 | LCM M2 positions, spotlight allowlist, Tiger Team blocklist, NV config |
| `DefaultHomePagePostProcessor.kt` | 1084 | Pinned/rankable split, post-ranking fixup orchestration |
| `NonRankableHomepageOrderingUtil.kt` | ~450 | All post-ranking fixups, gap rules, final re-indexing |
| `NvCarouselReplacementHandler.kt` | ~150 | NV carousel replacement/drop rules |
| `Ranking.kt` | 296 | sortOrder assignment from boost algorithm output |
| `EntityRankerConfiguration.kt` | 849 | LCM M2 intra-ranking, scoring |
