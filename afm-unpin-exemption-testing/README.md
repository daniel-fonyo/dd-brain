# AFM Unpin Exemption Testing

**PR under test**: [#62270 — Exempt AFM carousels from homepage NV carousel unpinning](https://github.com/doordash/feed-service/pull/62270)
**Reference PR**: [#61611 — Original AFM vertical exempt](https://github.com/doordash/feed-service/pull/61611)
**Author**: sri-yogesh-dorbala
**Date**: 2026-03-26

## Context

Adoption dropped ~11% over the past week because AFM carousels are being "unboosted" by the NV carousel unpinning logic. They should stay in the "two boost" category but are falling into "unboost". This is urgent — potential cherry-pick/emergency deploy.

## What the PR Does

PR #62270 refines the original AFM unpin exemption (#61611):

1. **Tighter AFM detection** — requires **both**:
   - First store has `AFFORDABLE_MEALS_VERTICAL_ID` (`100322L`) in `primaryVerticalIds`
   - First store has an active meal-box campaign (`isAffordableMealsMealBoxCampaign` — checks for `AFFORDABLE_MEALS_MEAL_BOX_CAMPAIGN` tag in `feedCampaigns`)
   - Carousel with AFM vertical but **no active campaign** still gets unpinned (tighter than #61611)

2. **DV gate** — `enable_afm_unpin_exemption` (boolean, default `false`). Safe rollback path.

### Files Changed
- `PersonalizationRuntimeUtil.kt` — new `enableAfmUnpinExemption()` DV reader
- `Boosting.kt` — refactored `isEligibleForUnpin()` + new `isAfmCarousel()` helper
- `BoostingTest.kt` — 3 new tests

## Carousels Under Test

| Carousel Name | Type | Expected Behavior When Pinned |
|---|---|---|
| **Shop the Meal Box** | Item carousel | Appears near **top** of homepage (gets boosted/ranked higher) |
| **Best $12 meals in Miami** | Store carousel | Appears **lower** on page but at configured sort order |

- Only one item carousel typically shows vs multiple store carousels
- Item carousel naturally ranks higher even when re-ranked by ML (user-dependent)

## Test Environment

- **Address**: `1111 Brickell Ave, Miami` — puts test Cx in AFM-eligible market
- **Domain**: `https://www.doordashtest.com/`
- **Credentials**: `tas-cx-doortest-egsn7vrtcl@doordash.com` / `1XQlCDr8Qb`
- **Sandbox sync**: `devbox run web-group1-remote` from PR branch working directory
- **Consumer ID override**: `757606047L` in `HomepageRequestToContext.kt`
