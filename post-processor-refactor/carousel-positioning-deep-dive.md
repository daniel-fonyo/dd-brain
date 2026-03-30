# Carousel Positioning: Complete Deep Dive

**Date:** 2026-03-30
**Purpose:** Single reference doc for how carousel positioning works, what's broken, and the path to a unified config. Supersedes `boost-pin-allowlist-analysis.md`, `carousel-positioning-audit.md`, and `unified-positioning-implementation-plan.md`.

---

## The Stakeholder Ask

> "Can we maintain a single source of truth of allowlist, that would power boosting or pinning by carousel ID, regardless of their vertical id? The existing ones can be reused but you will likely need to clean them up as they have tons of legacy NV ones that we don't want to boost any more."

---

## How Positioning Works Today

There are two fundamentally different mechanisms. They are NOT interchangeable.

### Pinning — exact slot, no ML

**Code:** `PinnedCarouselUtil.kt` L37-99, called from `DefaultHomePagePostProcessor.kt` L766-814

- Before ML scoring, `splitPinnedAndRankCarousels()` partitions carousels into **pinned** and **rankable**
- Pinned carousels are identified by matching their ID against `pinned_carousel_ranking_order.json`
- They get `sortOrder = 0, 1, 2...` based on their index in the JSON list — **exact slot assignment**
- **ML never touches them** — no Sibyl score, no ranking
- Used for: hero banner, search bar, fixed top-of-page carousels that must never move

### Boosting — ceiling guarantee, ML refines within range

**Code:** `Boosting.kt` L72-331, called from `EntityRankerConfiguration.kt` L504-510

- Carousels go through ML scoring first (Sibyl prediction score)
- `Boosted(position=N)` means: **guaranteed in the top N+1 slots, ML score determines the exact slot within that range**
- The 4-pass algorithm pools all carousels targeting the same ceiling together with unboosted carousels that fit in the available flowing slots, then **sorts the entire pool by ML score**
- A carousel boosted to position 4 could end up at position 0 (if it has the best ML score) or position 4 (if 4 other carousels outscore it). It will **never** be below position 4.
- Used for: NV carousels (Grocery, Convenience), taste carousels — anything that should be "near the top" with ML refinement

**Example from code comment (Boosting.kt L56-70):**
- 2 carousels boosted to k=3, 1 boosted to k=6
- For k=3: those 2 boosted + 1 best unboosted → sorted by ML score → positions 0, 1, 2
- For k=6: that 1 boosted + 2 best remaining unboosted → sorted by ML score → positions 3, 4, 5
- Rest: all remaining unboosted by ML score

### Why both exist — genuinely different needs

| | Pinning | Boosting |
|---|---|---|
| Position guarantee | Exact slot (position 0 = position 0) | Ceiling (position 4 = somewhere in top 5) |
| ML scored? | No | Yes |
| ML influences placement? | No | Yes — within the ceiling range |
| Runtime override possible? | No (except DV-gated list swaps) | Yes — NV unpin can strip boost |
| Use case | Must-be-here content (hero, search) | Should-be-near-top content (Grocery, taste) |

### ICB — Impression Capped Boosting

**Code:** `ImpressionCappedBoostingService.kt` L1-90

A wrapper around boosting. Pins NV carousels to a specific ceiling position for a capped number of impressions, then stops. Sets `enforceManualSortOrder=true` and `sortOrder=boostedPosition` on the carousel, which feeds into the normal boost algorithm.

- Matches by carousel **title** (not ID) — gap for unified config migration
- Currently active for "Grocery" and "Local liquor stores"
- Follows **boosting semantics** — ceiling guarantee, not exact slot

### The Boost Hint System

Each carousel type has `getBoostingHint()` that reads CMS fields:

| Type | Returns `Boosted(position)` when | File:Line |
|---|---|---|
| StoreCarousel, ItemCarousel, MapCarousel | `sortOrder != null` AND `enforceManualSortOrder == true` | L494-701 |
| StoreCollection, CollectionV2, ItemCollection, ReelsCarousel | `enforceManualSortOrder == true` (no sortOrder check) | L446-651 |
| DealCarousel, StoreEntity | Never — always `Unboosted` | L436, L661 |

`enforceManualSortOrder` is set by CMS catalog config, taxonomy tags, or programmatically by ICB.

### The NV Unpin Problem

**Code:** `Boosting.kt` L131-160

After boost hints are generated, NV unpin strips boost from non-Restaurant carousels (`verticalId != 1`). The `boostByPositionAllowList` is **not checked** — a carousel can be in the allowlist and still get unpinned. This is the AFM bug (see `afm-unpin-bug-and-boost-cleanup.md`).

---

## The 5 Config Sources

No single source of truth today. Positioning is determined by 5 independent systems:

| # | Config Source | What It Controls | Location |
|---|---|---|---|
| 1 | `pinned_carousel_ranking_order.json` (5 variants) | Which carousels get exact positions before ML | `PinnedCarouselUtil.kt` |
| 2 | `boost_by_position_carousel_id_allow_list.json` | Which carousels are eligible for ceiling-boost | `PersonalizationRuntimeUtil.kt` L236-242 |
| 3 | `carousel_capping_blocklist.json` | Which carousels are exempt from impression capping | `PersonalizationRuntimeUtil.kt` L317-323 |
| 4 | ICB config (DV experiment `530621f9-...`) | NV carousels eligible for impression-capped boosting | `ImpressionCappedBoostingService.kt` |
| 5 | Post-ranking fixups (hardcoded) | NV post-checkout, PAD, member pricing overrides | `NonRankableHomepageOrderingUtil.kt` |

---

## Pipeline Execution Order

A carousel's `sortOrder` is written up to 13 times. Last write wins.

```
Phase 0: CMS/Catalog sets sortOrder + enforceManualSortOrder
  ↓
Phase 1: PinnedCarouselUtil splits pinned (exact slot 0,1,2...) vs rankable
  ↓
Phase 2: ML Ranking
  ├─ Boost allowlist gates entry to boost algorithm
  ├─ ICB sets sortOrder on eligible NV carousels
  ├─ getBoostingHint() reads CMS fields → Boosted(ceiling) or Unboosted
  ├─ NV unpin strips boost from non-Restaurant (AFM bug here)
  ├─ 4-pass algorithm: reserves ceiling positions, ML refines within range
  └─ Score multiplier fabrication for downstream sort-by-score
  ↓
Phase 3: Dedup + Trim (position-sensitive — higher-ranked carousels win)
  ↓
Phase 4: Post-ranking fixups (last write wins):
  ├─ NV post-checkout → positions 0,1,2
  ├─ PAD → position 3
  ├─ Member pricing → position 0
  ├─ NV carousel replacement
  ├─ Spotlight banner position
  ├─ Color bleed reorder
  └─ Gap rules + immersive spacing
  ↓
FINAL: Re-indexing → mapIndexed to 0,1,2,3... (the only truth)
```

---

## Complete Config & Decision Inventory

### Runtime JSON Configs (7)

| File | Read By | Controls |
|---|---|---|
| `carousels/pinned_carousel_ranking_order.json` | `PinnedCarouselUtil` | Base pinned carousel order |
| `carousels/pinned_carousel_ranking_order_faraway.json` | `PinnedCarouselUtil` | Faraway intent pinned order |
| `carousels/pinned_carousel_ranking_order_nearby_rotation_v1.json` | `PinnedCarouselUtil` | Nearby rotation pinned order |
| `boost_by_position_carousel_id_allow_list.json` | `PersonalizationRuntimeUtil` | Boost eligibility |
| `carousel_capping_blocklist.json` | `PersonalizationRuntimeUtil` | Capping exemption |
| `carousel_capping_redisplay_probability.float` | `PersonalizationRuntimeUtil` | Resurfacing probability |
| `lcm_carousels_vertical_positions_by_treatment_arm.json` | `DiscoveryRuntimeUtil` | LCM M2 positions per treatment |

### DV Experiment Gates (18)

| DV / Experiment | Controls | Used At |
|---|---|---|
| `discovery_p13n_unpin_bookmarks` | Remove carousel IDs from pinned list | `PinnedCarouselUtil` L177 |
| `GiftsExperimentManager.TREATMENT_PINNED` | Use faraway pinned list | `PinnedCarouselUtil` L123 |
| `PADUtil.shouldUnpinTasteOfDPCollection()` | Unpin Taste of DashPass | `PinnedCarouselUtil` L140 |
| `PADUtil.isAnyNearbyRotationV1Treatment()` | Use nearby rotation pinned list | `PinnedCarouselUtil` L128 |
| `shouldUpdateOffersForYouSortOrder()` | Deal carousel random pin within top 5 | `DHPP` L824 |
| `enableHomepageVerticalBlending` | Gate NV unpin logic | `Boosting.kt` L135 |
| `ENABLE_AFM_UNPIN_EXEMPTION` | Exempt AFM from NV unpin | `Boosting.kt` L745 |
| `TigerTeamFeature.CONTENT_SYSTEM_BOOSTING` | Override boost allowlist to `dt:` only | `Boosting.kt` L90 |
| `getDealCarouselScoreMultiplier()` | Multiply deal carousel score | `Boosting.kt` L176 |
| `IMPRESSION_CAPPED_BOOSTING_FEATURE_FLAG` | Gate ICB system | `ImpressionCappedBoostingService` |
| `ENABLE_LCM_M2_USE_CASE_CAROUSELS` | Gate LCM M2 positioning | `ImpressionCappedBoostingService` L223 |
| `ENABLE_CAROUSEL_CAPPING` | Gate capping + threshold | `PersonalizationExperimentManager` L243 |
| `UNIVERSAL_RANKER_V5/FW/FW_DELAY` | Override capping for UR holdouts | `PersonalizationExperimentManager` L252-261 |
| `TigerTeamFeature.DISABLE_THE_CAROUSEL_CAPPING` | Exempt non-DT from capping | `CarouselCapping.kt` L334 |
| `isPostCheckoutNVContentBoosting` | NV post-checkout pin to 0,1,2 | `NonRankableHomepageOrderingUtil` L269 |
| `PAD_CAROUSEL_SPOTLIGHT_V2` | PAD → position 3 | `NonRankableHomepageOrderingUtil` L300 |
| `isMemberPricingV2PriorityCarouselsEnabled()` | Member pricing → position 0 | `NonRankableHomepageOrderingUtil` L230 |
| `shouldEnableNvCarouselReplacements()` | NV carousel swap/drop | `NvCarouselReplacementHandler` L36 |

### Hardcoded Values (6)

| Value | What | File:Line |
|---|---|---|
| `RESTAURANT_VERTICAL_ID = 1` | NV unpin check | `Boosting.kt` L725 |
| `AFFORDABLE_MEALS_VERTICAL_ID = 100322` | AFM exemption | `Boosting.kt` L760 |
| `DEAL_CAROUSEL_PIN_RANGE = 5` | Random pin range for deal | `DHPP` L94 |
| PAD position = `3` | PAD on treatment1 | `NonRankableHomepageOrderingUtil` L311 |
| Member pricing position = `0` | Member pricing override | `NonRankableHomepageOrderingUtil` L258 |
| NV post-checkout = `0, 1, 2` | Top 3 NV positions | `NonRankableHomepageOrderingUtil` L276-289 |

### sortOrder Write Sites (13, chronological)

| # | When | What | Value | File:Line |
|---|---|---|---|---|
| 1 | Data fetching | Operator collection ordering | `operatorOverrideOrder` | `ItemCarouselOperatorCollectionContentFetching.kt` L357 |
| 2 | ICB application | ICB boost applied | `boostedPosition` | `ImpressionCappedBoostingService.kt` L146 |
| 3 | Pinned split | Pinned carousels ordered | `0, 1, 2...` per rule list | `PinnedCarouselUtil.kt` L37-99 |
| 4 | Pinned split | Rankable pushed | `+= pinnedCount` | `DHPP.mergeContentWithPushedSortOrder()` |
| 5 | LCM M2 | LCM treatment positions | Config-driven | `EntityRankerConfiguration.kt` L723 |
| 6 | Boost algorithm | 4-pass ceiling reservation | Positioner result | `Ranking.kt` L126-134 |
| 7 | Post-dedup | NV post-checkout | `0, 1, 2` | `NonRankableHomepageOrderingUtil.kt` L276-289 |
| 8 | Post-dedup | PAD spotlight | `3` | `NonRankableHomepageOrderingUtil.kt` L311 |
| 9 | Post-dedup | Member pricing | `0` | `NonRankableHomepageOrderingUtil.kt` L258 |
| 10 | Post-dedup | Spotlight banner | `0` or config | `NonRankableHomepageOrderingUtil.kt` L176 |
| 11 | Post-dedup | Color bleed reorder | Shifted | `NonRankableHomepageOrderingUtil.kt` L97-104 |
| 12 | Post-dedup | Gap rules | Shifted | `GapRulesUtil.reorderForImmersiveContentSpacing()` |
| **13** | **FINAL** | **Re-indexing** | **`0,1,2,3...`** | **`NonRankableHomepageOrderingUtil.kt` L113-115** |

---

## Spotlight Carousels

Despite stakeholder concern, spotlight carousels do **not** have hardcoded pinning that bypasses the allowlist:

- SpotlightBanner and CollectionV2 items use standard `sortOrder`
- Merged into the general list and sorted alongside everything else
- A spotlight migration allowlist gates **visibility**, not position
- Spotlights **can** appear in `pinned_carousel_ranking_order.json` like any other carousel
- Feed placement blending (M2/M3 modes) slots spotlights at positions 2, 5, 8 via hash-based logic
- `SpotlightClashUtil.designatePrioritySpotlight()` keeps only one when multiple compete

**Open question:** Confirm with Grant Roberts whether spotlight positions are CMS-configured upstream or ML-ranked. If CMS sets a fixed `sortOrder`, that's the "hardcoded" logic the stakeholder means — it's upstream, not in our code.

---

## Unified Config: What It Looks Like

A unified config needs **both strategies** because pinning and boosting are fundamentally different:

```json
{
  "carousels": [
    {
      "carousel_id": "hero-banner-id",
      "position": 0,
      "strategy": "PIN",
      "notes": "Exact slot 0, no ML"
    },
    {
      "carousel_id": "taste:grocery",
      "position": 4,
      "strategy": "BOOST",
      "notes": "Ceiling = top 5, ML picks exact slot within"
    },
    {
      "carousel_id": "afm-store-carousel-id",
      "position": 3,
      "strategy": "BOOST",
      "protect_from_unpin": true,
      "notes": "Affordable Meals — verticalId=100322 but should be boosted"
    }
  ],
  "defaults": {
    "nv_carousels": "ML_RANKED"
  }
}
```

### What each strategy means

| Strategy | Behavior | Replaces |
|---|---|---|
| `PIN` | Exact slot. Split out before ML. No ML score. | `pinned_carousel_ranking_order.json` |
| `BOOST` | Ceiling guarantee. ML scored + ML refines within range. | `boost_by_position_allow_list` + `enforceManualSortOrder` |
| `ML_RANKED` | Pure ML. No position override. | Default (no config) |

### What this eliminates

1. NV unpin by vertical ID — if a carousel is in the config with `BOOST`, it's protected. Period.
2. 5 separate configs → 1 file declares intent
3. Stale NV entries — absent from config = `ML_RANKED`
4. `enforceManualSortOrder` CMS dependency for boosted carousels — position comes from config
5. The AFM bug category — config is the authority, not vertical ID

### Prerequisites for unification

1. **ICB must migrate from title matching to ID matching** — currently matches "Grocery" not carousel ID
2. **Post-ranking fixups must become data-driven** — PAD=3, member pricing=0, NV post-checkout=0,1,2 are hardcoded
3. **Stale entries must be cleaned** — Parul's SOT audit of what's intentionally boosted/pinned today
4. **`enforceManualSortOrder` inventory** — which carousels have this set upstream by CMS

---

## Implementation: Option A (Standalone, Before RFC)

### Approach

DV-gated with shadow A/A validation. Old path always runs. New path runs in parallel on a deep copy. Shadow compares outputs; serving ramps only after 100% match.

### New Files (3)

1. **`CarouselPositioningConfig.kt`** — Data model + runtime reader. `findRule(carouselId)` returns position rule or null. Exact match > prefix match (for `taste:`, `rt:`).
   - Path: `libraries/domain-util/.../ranking/CarouselPositioningConfig.kt`

2. **`carousel_positioning_config.json`** — Runtime JSON populated from existing configs + Parul's SOT.
   - Path: `carousels/carousel_positioning_config.json`

3. **`UnifiedPositioningShadow.kt`** — Compares old vs new `RankableContent` outputs. Emits per-carousel position deltas, sortOrder match, carousel count match.
   - Path: `libraries/domain-util/.../ranking/UnifiedPositioningShadow.kt`

### Modified Files (2)

1. **`DefaultHomePagePostProcessor.rankAndMergeContent()`** (L708-759) — Adds DV-gated fork. Old path extracted to `oldRankAndMergeContent()`. New path sends ALL carousels through `rankContent()` (no pinned split). Shadow mode: both run, compare, serve old. Serving mode: new path serves.

2. **`BoostingBundle.boosted()`** (L72-160) — Adds `useUnifiedPositioning` parameter. When true: reads `CarouselPositioningConfig` instead of allowlist + `getBoostingHint()`. No NV unpin block. Config entries → `Boosted(ceiling)`, absent → `Unboosted`. The 4-pass algorithm (L162-331) is **completely unchanged** — it just receives differently-populated `toBoostPairs`/`unBoostPairs`.

### Call chain

```
DHPP.rankAndMergeContent()
  ├─ OLD: splitPinnedAndRankCarousels() → rankContent() → mergeContentWithPushedSortOrder()
  └─ NEW: rankContent(deepCopy)  [no split — all carousels enter]
           → EntityRankerConfiguration.getBoostBundle()
             → BoostingBundle.boosted(useUnifiedPositioning=true)
               → CarouselPositioningConfig.findRule(id) → Boosted(ceiling) or Unboosted
               → NO unpin, NO allowlist, NO enforceManualSortOrder
               → 4-pass algorithm runs identically
```

### Key risks

| Risk | Mitigation |
|---|---|
| Dedup divergence (pinned carousels now have ML scores, different dedup walk) | Shadow `store_dedup_match` metric. Ensure config positions match pinned list order. |
| Deal carousel random pinning diverges from ML ranking | Add deal carousel to config, or accept as intentional improvement |
| ICB carousels not in unified config | Populate config with ICB carousel IDs during setup |
| Post-ranking fixups still overwrite unified positions | Unchanged in Phase 1. Config-driven fixups are Phase 2. |

### PR breakdown (~290 lines total)

| PR | What | Size |
|---|---|---|
| 1 | Config model + reader + runtime JSON | ~150 lines |
| 2 | `BoostingBundle.boosted()` unified branch | ~60 lines |
| 3 | `DHPP.rankAndMergeContent()` shadow/serving | ~80 lines |
| 4 | DV setup + dashboard + shadow enable at 1% | Config only |

---

## Recommended Approach (Sequenced)

**Critical constraint:** Option 2 fix (allowlist protects against NV unpin) must NOT ship before allowlist cleanup. Stale NV entries would be re-pinned.

| Step | What | When | Blocks |
|---|---|---|---|
| **A** | Parul's SOT audit — which carousels need boosting/pinning today | This week | C |
| **B** | Confirm spotlight positioning with Grant Roberts | This week | C |
| **C** | Clean up existing configs — remove stale NV entries | After A+B | D |
| **D** | AFM fix — allowlist protects against unpin (`Boosting.kt` L139) | After C | — |
| **E** | Ranking Pipeline RFC M1-4 — step scaffold | Parallel to A-D | F |
| **F** | Option A — unified config with shadow A/A | After C | — |
| **G** | Post-processor RFC — `FinalOverwriteStep` absorbs all configs | After E+F | E, F |

---

## Open Questions

- [ ] Grant Roberts: How are spotlight carousel positions determined? CMS-configured or ML-ranked?
- [ ] Parul: SOT of all current intentional pinning/boosting needs
- [ ] ICB team: Can ICB config migrate from title matching to ID matching?
- [ ] Are there other teams setting `sortOrder` upstream that we don't know about?
- [ ] For the unified config, should `BOOST` carousels that are currently pinned (exact slot) keep exact slot behavior, or is ceiling-with-ML acceptable? This changes dedup behavior.

---

## Key Files Reference

| File | Lines | What |
|---|---|---|
| `Boosting.kt` | 765 | Boost algorithm, allowlist, NV unpin, AFM exemption, 4-pass ceiling reservation |
| `PinnedCarouselUtil.kt` | 183 | Pinned list construction, JSON variant selection, DV filtering |
| `CarouselCapping.kt` | 419 | V1/V2 capping, blocklist, ICB exemption |
| `ImpressionCappedBoostingService.kt` | 247 | ICB system, M2 scoring |
| `PersonalizationRuntimeUtil.kt` | ~400 | Boost allowlist reader, capping blocklist reader |
| `DiscoveryRuntimeUtil.kt` | ~3000 | LCM M2 positions, spotlight allowlist, NV config |
| `DefaultHomePagePostProcessor.kt` | 1084 | Pinned/rankable split, post-ranking fixup orchestration |
| `NonRankableHomepageOrderingUtil.kt` | ~450 | All post-ranking fixups, gap rules, final re-indexing |
| `EntityRankerConfiguration.kt` | 849 | ML scoring, boost bundle construction |
| `Ranking.kt` | 296 | sortOrder assignment from boost output |
