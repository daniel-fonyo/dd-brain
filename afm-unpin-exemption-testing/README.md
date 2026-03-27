# AFM Unpin Exemption Testing

**PR under test**: [#62270 — Exempt AFM carousels from homepage NV carousel unpinning](https://github.com/doordash/feed-service/pull/62270)
**Reference PR**: [#61611 — Original AFM vertical exempt](https://github.com/doordash/feed-service/pull/61611)
**Author**: sri-yogesh-dorbala
**Date**: 2026-03-26

## What the PR Does

PR #62270 refines the original AFM unpin exemption (#61611) with two key changes:

1. **Tighter AFM detection**: Instead of exempting *all* carousels with `AFFORDABLE_MEALS_VERTICAL_ID`, it now requires **both**:
   - First store has `AFFORDABLE_MEALS_VERTICAL_ID` in `primaryVerticalIds`
   - First store has an active AFM meal-box campaign (`isAffordableMealsMealBoxCampaign` on `feedCampaigns`)

   A carousel with the AFM vertical but **no active campaign** will still be unpinned.

2. **DV gate**: `enable_afm_unpin_exemption` (boolean, default `false`). The exemption is only active when this DV is `true`. When `false`, all NV carousels (including AFM) are unpinned as before.

### Files Changed
- `PersonalizationRuntimeUtil.kt` — new `enableAfmUnpinExemption()` DV reader
- `Boosting.kt` — refactored `isEligibleForUnpin()` + new `isAfmCarousel()` helper
- `BoostingTest.kt` — 3 new tests (DV on + campaign, DV off, DV on but no campaign)

## Test Matrix

| # | Scenario | DV State | AFM Campaign | Expected Behavior | Test Method |
|---|----------|----------|-------------|-------------------|-------------|
| 1 | AFM carousel with active campaign, DV ON | `true` | present | AFM carousel **stays pinned** at configured position | Unit + Sandbox |
| 2 | AFM carousel with active campaign, DV OFF | `false` | present | AFM carousel **unpinned** (re-ranked by ML) | Unit + Sandbox |
| 3 | AFM-vertical carousel, no campaign, DV ON | `true` | absent | Carousel **unpinned** (not recognized as AFM) | Unit + Sandbox |
| 4 | Regular NV carousel (non-AFM vertical), DV ON | `true` | N/A | Carousel **unpinned** (not AFM) | Unit |
| 5 | Restaurant carousel, DV ON | `true` | N/A | Carousel **stays pinned** (restaurant exempt by default) | Unit |
| 6 | Item carousel with AFM vertical + campaign, DV ON | `true` | present | Item carousel **stays pinned** | Unit |

## Testing Plan

### Phase 1: Unit Test Verification
- **Status**: Covered in PR (scenarios 1-3)
- Run `BoostingTest` locally to confirm all pass
- Verify edge cases: null stores, empty vertical lists, empty campaigns

### Phase 2: Sandbox E2E Testing
Deploy PR branch to sandbox and validate real homepage behavior.

#### Pre-requisites
1. Sync PR branch code to sandbox via `devbox run web-group1-remote`
2. Apply personal sandbox overrides (hardcoded consumer ID, Iguazu bypass if needed)
3. Confirm AFM carousel exists for test consumer on homepage

#### Test Steps

**Test 2A: DV OFF (baseline — current prod behavior)**
1. Ensure `enable_afm_unpin_exemption` DV = `false`
2. Load homepage on sandbox (`doordashtest.com`)
3. Observe AFM carousel position — should be **re-ranked by ML** (not pinned)
4. Screenshot + log carousel ordering

**Test 2B: DV ON — AFM carousel with active campaign**
1. Set `enable_afm_unpin_exemption` DV = `true`
2. Reload homepage
3. Observe AFM carousel position — should be **pinned at configured sort order**
4. Compare position to Test 2A — should be different (higher/pinned)
5. Screenshot + log carousel ordering

**Test 2C: Verify non-AFM NV carousels still unpinned**
1. With DV = `true`
2. Check other NV carousels (e.g., Grocery, Convenience) — should still be **unpinned/re-ranked**
3. Screenshot

### Phase 3: Production Verification (post-merge)
1. DV rolled out gradually
2. Monitor homepage carousel ordering for AFM carousels
3. Confirm no regressions for other NV carousels

## Open Questions
- What consumer ID / address sees AFM carousels reliably on sandbox?
- Is there a way to force-inject a feed campaign for testing scenario 3 (vertical but no campaign)?
- Should we also validate Iguazu events show correct carousel positions?
