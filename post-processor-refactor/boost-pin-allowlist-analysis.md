# Carousel Boosting & Pinning: Single Source of Truth Analysis

**Date:** 2026-03-30
**Context:** Stakeholder ask — can we maintain a single source of truth allowlist that powers boosting or pinning by carousel ID, regardless of vertical ID? Clean up legacy NV entries. Account for spotlight carousels.
**Relates to:** [Ranking Pipeline RFC](./RFC.md), [AFM Unpin Bug](./afm-unpin-bug-and-boost-cleanup.md), UBP

---

## The Ask (Verbatim)

> "I think this functionality is also a critical one for the UBP that you are driving. Currently the status is that if it is NV then disable; otherwise, respect the boosting and pinning, and we also have spotlight carousels being pinned (ask your buddy Grant Roberts), which might have a hard-coded logic that doesn't even go through our allowlist. My ask to you is that: can we maintain a single source of truth of allowlist, that would power boosting or pinning by carousel ID, regardless of their vertical id, or any logic that you see fit. The existing ones can be reused but you will likely need to clean them up as they have tons of legacy NV ones that we don't want to boost any more."

---

## Current State: 5 Separate Config Sources Control Positioning

There is no single source of truth today. Carousel positioning is determined by **5 independent config/code sources** that interact in non-obvious ways:

| # | Config Source | What It Controls | Format | Location |
|---|---|---|---|---|
| 1 | **`pinned_carousel_ranking_order.json`** | Which carousels get fixed positions (slots 0, 1, 2...) BEFORE ML ranking | Ordered list of carousel IDs | `PinnedCarouselUtil.kt` reads from runtime JSON |
| 2 | **`boost_by_position_carousel_id_allow_list.json`** | Which carousels are eligible for the 4-pass boosting algorithm (position boosting within ML-ranked carousels) | Set of carousel IDs (or `"ALL"`) | `PersonalizationRuntimeUtil.kt` L236-242 |
| 3 | **`carousel_capping_blocklist.json`** | Which carousels are exempt from impression capping (always shown) | Set of carousel IDs | `PersonalizationRuntimeUtil.kt` L317-323 |
| 4 | **ICB config (DV experiment)** | NV carousels eligible for impression-capped boosting (pin for N impressions then stop) | `ImpressionCappedBoostingConfig` with entity titles | DV experiment `530621f9-...` |
| 5 | **Post-ranking fixups** (hardcoded) | NV post-checkout pinning, PAD pinning, member pricing override | Hardcoded positions in code | `DefaultHomePagePostProcessor.kt` fixups 1-5 |

### How They Interact (Execution Order)

```
REQUEST
  │
  ├─ 1. PinnedCarouselUtil reads pinned_carousel_ranking_order.json
  │     → Carousels in list get fixed positions 0, 1, 2...
  │     → Everything else goes to ML ranking
  │
  ├─ 2. ML Ranking (EntityRankerConfiguration)
  │     ├─ boost_by_position_allow_list gates entry into boosting algorithm
  │     ├─ NV unpin (Boosting.kt L131-148) strips boost from non-Restaurant verticals
  │     │   ⚠ THIS IS WHERE THE AFM BUG LIVES — allowlist is ignored
  │     ├─ ImpressionCappedBoosting pins NV carousels for capped impressions
  │     └─ carousel_capping_blocklist exempts carousels from capping
  │
  ├─ 3. Dedup + Trim
  │
  └─ 4. Post-ranking fixups (run IN THIS ORDER, last write wins):
        ├─ updateSortOrderOfNVCarousel() — post-checkout NV boost to positions 0,1,2
        ├─ updateSortOrderOfTasteOfDashPass() — PAD hardcoded to position 3
        ├─ updateSortOrderOfMemberPricing() — member pricing hardcoded to position 0
        ├─ NV carousel replacements
        └─ Gap rules + final re-indexing
```

**The fundamental problem:** A carousel's final position is determined by the union of all 5 systems, each with its own config and none aware of the others. If someone adds a carousel to `pinned_carousel_ranking_order.json` AND it's in the `boost_by_position_allow_list` AND ICB is active for it, the interactions are undefined.

---

## Spotlight Carousels: No Hardcoded Bypass Found

Despite the stakeholder concern, spotlight carousels do **not** have hardcoded pinning that bypasses the allowlist:

- **SpotlightBanner** and spotlight **CollectionV2** items use the standard `sortOrder` property
- They are merged into the general carousel list and sorted by `sortOrder` alongside everything else
- A **spotlight migration allowlist** (`occasion/spotlight_migration_allowlist.json`) gates which spotlight campaigns are allowed to appear, but this controls visibility, not position
- No Grant Roberts-specific hardcoded logic found in the codebase
- Spotlight carousels CAN appear in `pinned_carousel_ranking_order.json` like any other carousel

**Conclusion:** Spotlight positioning is already governed by the same system as everything else. The ask to "go through our allowlist" is already satisfied — spotlight just uses `sortOrder` + the pinned carousel config.

**Action item:** Confirm with Grant Roberts that there isn't an external system (CMS config, campaign config) setting spotlight `sortOrder` to a fixed value outside of feed-service code. If there is, that's the "hardcoded" logic the stakeholder is referring to — it would be upstream, not in our codebase.

---

## What a Single Source of Truth Looks Like

### Proposed: Unified Carousel Positioning Config

One config that declares the **intent** for each carousel — what positioning treatment it should receive:

```json
{
  "version": 2,
  "carousels": {
    "favorites": {
      "treatment": "pinned",
      "position": 0,
      "notes": "Always first — user's saved favorites"
    },
    "reorder": {
      "treatment": "pinned",
      "position": 1,
      "notes": "Recent orders — high conversion"
    },
    "afm-store-carousel-id-here": {
      "treatment": "boosted",
      "position": 3,
      "protect_from_unpin": true,
      "notes": "Affordable Meals — Rx-like but verticalId=100322"
    },
    "gen-ai-carousel-id-here": {
      "treatment": "boosted",
      "position": 5,
      "protect_from_unpin": true,
      "notes": "Gen-AI personalized carousel"
    },
    "taste:*": {
      "treatment": "ml_ranked",
      "capping_exempt": false,
      "notes": "Taste carousels — ML decides position, subject to capping"
    }
  },
  "defaults": {
    "nv_carousels": {
      "treatment": "ml_ranked",
      "capping_exempt": false,
      "notes": "NV carousels freely ranked by ML unless explicitly listed above"
    }
  }
}
```

### Treatment Types

| Treatment | Meaning | Replaces |
|---|---|---|
| `pinned` | Fixed position, ML cannot move it | `pinned_carousel_ranking_order.json` |
| `boosted` | Enters boosting algorithm at configured position, protected from NV unpin | `boost_by_position_allow_list` + unpin protection |
| `ml_ranked` | ML freely ranks, no position override | Default (no config needed) |
| `icb` | Impression-capped boost — pinned for N impressions then ML-ranked | `ImpressionCappedBoostingConfig` |
| `capping_exempt` | Never impression-capped (always shown) | `carousel_capping_blocklist` |

### What This Eliminates

1. **No more vertical ID logic for unpin** — if a carousel is `boosted` or `pinned`, it's protected. Period. The AFM bug category disappears.
2. **No more 5 separate configs** — one file declares intent; the pipeline reads it once.
3. **Legacy NV cleanup** — carousels not in the config get `ml_ranked` by default. Stale NV entries from old experiments are simply absent.
4. **Spotlight covered** — spotlight carousels can be `pinned` or `boosted` like anything else.

---

## How Our Proposals Support This

### Phase 1: Short-term fix (Option 2 from AFM analysis) — NOW

Wire `boostByPositionAllowList` into the unpin filter at `Boosting.kt` L139. This is a one-line code change that makes the existing allowlist protect against unpin.

**What it does for the SOT ask:** Makes `boost_by_position_allow_list` the de facto authority for "should this carousel keep its boost?" Doesn't unify the 5 configs but makes the most important one (boost allowlist) actually work correctly.

**Effort:** Hours. No cross-team coordination needed.

### Phase 2: Allowlist cleanup — 1-2 WEEKS

1. Parul provides current SOT of all intentional pinning/boosting needs
2. Remove legacy NV entries from `boost_by_position_allow_list`
3. Remove stale entries from `pinned_carousel_ranking_order.json`
4. Document which carousels are in which config and why

**What it does for the SOT ask:** Cleans up the existing configs so they're accurate and trusted. Still 5 separate configs, but each is clean.

### Phase 3: Ranking Pipeline RFC (Milestones 1-4) — 5-7 WEEKS

The [Ranking Pipeline RFC](./RFC.md) wraps the existing ranking engine in a step-based pipeline. Position boosting becomes `BoostBundleStep` — an addressable step rather than logic buried in a 258-line method.

**What it does for the SOT ask:** Creates the structural scaffold where a unified config can be injected. Doesn't change the config story yet, but makes it possible to change it without touching everything.

### Phase 4: Sub-step decomposition (Milestone 5) — SEPARATE RFC

`BoostBundleStep` decomposes into 6 independent sub-steps:
- `ScoreAssignmentSubStep`
- `LcmIntraRankingSubStep`
- `StoreListRerankingSubStep`
- `IntermixingSplitSubStep`
- `ImpressionCappingSubStep`
- `PositionBoostingSubStep`

All 31 DVs resolved once into `RankingPipelineState`. Every sub-step reads config from state, not from scattered static calls.

**What it does for the SOT ask:** `PositionBoostingSubStep` is the natural home for a unified positioning config. It reads one config, applies boosting according to declared intent, and the NV unpin logic either moves into `isApplicable()` or is eliminated entirely.

### Phase 5: Post-processor pipeline + FinalOverwriteStep — FUTURE RFC

Yu's north-star vision. The pipeline decomposes `reOrderGlobalEntitiesV2()` into 9 steps. The 5 post-ranking fixups (NV pinning, PAD, member pricing) become a single `FinalOverwriteStep`:

```
Steps 1-7:  ML ranking, placement blending, dedup, sort order adjustments
Step 8:     FinalOverwriteStep — reads unified config, pins carousels to declared positions
Step 9:     Gap rules + final ordering
```

**What it does for the SOT ask:** This IS the single source of truth. One config file. One step that reads it. Everything else is ML-ranked. The 5 separate configs collapse into one. The unified carousel positioning config from the proposal above becomes the input to `FinalOverwriteStep`.

---

## Gap Analysis: What Our Proposals DON'T Cover

### 1. ICB (Impression-Capped Boosting) integration

ICB is a separate system with its own config (`ImpressionCappedBoostingConfig`), its own service (`ImpressionCappedBoostingService`), and its own eligibility checkers. Our RFC treats it as part of `BoostBundleStep` but doesn't explicitly plan to unify its config with the boost allowlist.

**Gap:** The unified config needs an `icb` treatment type. ICB entity titles (currently matched by carousel *title*, not ID) need to migrate to carousel ID-based matching to be part of the unified allowlist.

**Effort to close:** Medium. ICB config migration is mostly mechanical but needs ICB team alignment.

### 2. `pinned_carousel_ranking_order.json` is a separate system

The pinned carousel list (`PinnedCarouselUtil`) runs BEFORE ML ranking. It's a completely separate code path from boosting. Our RFC wraps the ranking engine (which includes boosting) but the pinned-vs-rankable split happens upstream in `splitPinnedAndRankCarousels()`.

**Gap:** The unified config needs to handle both pre-ranking pinning AND post-ranking boosting. Today these are structurally different: pinned carousels never enter the ranking pipeline at all.

**How to close:** In `FinalOverwriteStep` (Phase 5), ALL carousels go through ML ranking. Pinning moves from pre-ranking split to post-ranking override. The pinned carousel list is absorbed into the unified config.

**Risk:** Changing from pre-ranking pinning to post-ranking override changes dedup behavior (dedup is position-sensitive). A carousel that was pinned at position 0 always won dedup. If it's now ML-ranked and then overridden, it might lose stores to dedup first. Need shadow comparison to validate.

### 3. Post-ranking fixups are hardcoded business logic

NV post-checkout pinning, PAD pinning, and member pricing override are hardcoded in `DefaultHomePagePostProcessor.kt`. They're not config-driven — they're code with DV gates.

**Gap:** These fixups need to become config-driven rules in the unified config, not hardcoded steps. Something like:

```json
{
  "conditional_overrides": [
    {
      "condition": "post_checkout_nv",
      "carousels": ["grocery-carousel-id"],
      "positions": [0, 1, 2],
      "dv_gate": "post_checkout_nv_boosting"
    },
    {
      "condition": "always",
      "carousels": ["member-pricing-carousel-id"],
      "positions": [0],
      "dv_gate": "member_pricing_v2_priority"
    }
  ]
}
```

**How to close:** This is `FinalOverwriteStep` scope. The fixups become rules in config rather than hardcoded methods.

### 4. Spotlight upstream config

Spotlight carousels don't have hardcoded pinning in feed-service, but their `sortOrder` may be set by an upstream CMS/campaign system. If the CMS hardcodes `sortOrder=2` for a spotlight, feed-service sees it as a pre-configured position — it's not visible in our allowlists.

**Gap:** Need to confirm with Grant Roberts whether spotlight positions are CMS-configured or ML-ranked. If CMS-configured, the unified config needs a way to either override or respect upstream positions.

**How to close:** If spotlight positions come from CMS, add them to the unified config as `pinned` with their CMS positions. If they're ML-ranked, no action needed.

---

## Recommended Approach

**Critical ordering constraint:** Option 2 (allowlist protects against unpin) must NOT ship before the allowlist is cleaned up. Today the allowlist contains stale NV carousel IDs that we don't want boosted. The current NV unpin logic accidentally neutralizes those stale entries — if we protect them from unpin before removing them, we'd be *re-pinning* carousels that should be ML-ranked. Cleanup first, then code change.

| Step | What | When | Blocks |
|---|---|---|---|
| **A** | Parul's SOT audit — which carousels actually need boosting/pinning today | **This week** | Step C |
| **B** | Confirm spotlight positioning with Grant Roberts | **This week** | Step C |
| **C** | Clean up existing configs — remove legacy NV entries from `boost_by_position_allow_list`, reconcile with Parul's SOT | **After A+B** (1-2 weeks) | Step D |
| **D** | Option 2 fix — allowlist protects against unpin (safe now that allowlist is clean) | **After C** (hours of code, but blocked on C) | Nothing |
| **E** | Ranking Pipeline RFC M1-4 — step scaffold | **5-7 weeks** | Nothing (parallel to A-D) |
| **F** | Milestone 5 — `PositionBoostingSubStep` with unified config input | **After E** | E |
| **G** | Post-processor RFC — `FinalOverwriteStep` absorbs all 5 configs | **After F** | F |

Steps A-D clean up the current system and establish a trusted allowlist. Steps E-G build the structural scaffold that makes a true single source of truth possible. The two tracks run in parallel.

**The honest answer to the stakeholder's ask:** Yes, we can maintain a single source of truth. Steps A-D give us a *clean* allowlist that works correctly (weeks). The RFC gives us the *architecture* where that allowlist becomes the only config needed (months). The two tracks are complementary.

---

## Key Files Reference

| File | What |
|---|---|
| `Boosting.kt` L69-148 | Boost algorithm: allowlist check, hint, NV unpin |
| `Boosting.kt` L693-714 | `isEligibleForUnpin()` — AFM bug location |
| `PinnedCarouselUtil.kt` | Pre-ranking pinned carousel split |
| `PersonalizationRuntimeUtil.kt` L236-242 | `boostByPositionCarouselIdAllowList()` |
| `PersonalizationRuntimeUtil.kt` L310-327 | Capping params + blocklist |
| `CarouselCapping.kt` | Impression capping + bypass logic |
| `ImpressionCappedBoostingService.kt` | ICB system |
| `DefaultHomePagePostProcessor.kt` L255-476 | Post-ranking fixups |
| `DiscoveryRuntimeUtil.kt` L1390 | Spotlight migration allowlist |

---

## Open Questions

- [ ] Grant Roberts: How are spotlight carousel positions determined? CMS-configured or ML-ranked?
- [ ] Parul: SOT of all current intentional pinning/boosting needs (non-blender)
- [ ] ICB team: Can ICB config migrate from entity-title matching to carousel-ID matching?
- [ ] Is the Sigma dashboard reliable for validating current pin/boost counts?
- [ ] Are there other teams setting `sortOrder` upstream that we don't know about?
