# AFM Carousel Unpin Bug & Boost/Pin Cleanup

**Date:** 2026-03-26
**Slack thread:** [#ask-p13n — Mar 18](https://doordash.slack.com/archives/C06GLRH4ACE/p1773851355961119)
**Participants:** Parul, Michael Chen, Yu Zhang, Daniel Fonyo
**Status:** Open — needs implementation

---

## The Bug

AFM (Affordable Meals) store + item carousels are getting **unintentionally unboosted** on the homepage.

**Root cause:** `Boosting.kt:693-714` — `isEligibleForUnpin()` strips boost/pin from any carousel whose first store's `primaryVerticalIds` doesn't include Restaurant (`ID: 1`). AFM carousels are functionally Rx but use `primaryVerticalId = 100322` (`AFFORDABLE_MEALS_VERTICAL_ID`), so they're treated as NV and unpinned. The ML model then re-ranks them down.

```kotlin
// Boosting.kt L693-714
private fun isEligibleForUnpin(candidate: BoostingCandidate): Boolean {
    val primaryVerticalIds = when (candidate) {
        is BoostingCandidate.StoreCarousel -> candidate.carousel.stores.firstOrNull()?.primaryVerticalIds
        is BoostingCandidate.ItemCarousel -> candidate.carousel.itemStoreEntities.firstOrNull()?.primaryVerticalIds
        else -> null
    }
    return primaryVerticalIds?.isNotEmpty() == true && !primaryVerticalIds.contains(RESTAURANT_VERTICAL_ID)
}
```

Only `StoreCarousel` and `ItemCarousel` candidates are affected — all other `BoostingCandidate` types return `null` and are never unpinned.

**Impact:** AFM carousels lose their configured boost positions and get ranked purely by the ML model, which was not intended.

---

## Why the Existing Allowlist Doesn't Help

The `boost_by_position_carousel_id_allow_list` runtime config exists and AFM carousel IDs are in it. But it only gates **entry** into boosting — it provides **zero protection** against the unpin step.

Code trace for an AFM carousel that IS in the allowlist:

1. **L90-98** — `boostByPositionAllowList` built from runtime config. AFM ID is in the set.
2. **L112** — `isBoostingEnabled(boostByPositionAllowList, afmId)` → `true` → enters boost logic.
3. **L122-125** — Hinter returns `Boosted(position)` (AFM has `enforceManualSortOrder=true` + `sortOrder`). **AFM goes into `toBoostPairs`.**
4. **L139-147** — NV unpin runs. `isEligibleForUnpin(afm)` → vertical ID 100322 ≠ 1 → `true`. **AFM is removed from `toBoostPairs` and moved to `unBoostPairs`.** The allowlist is not checked here.

Cleaning up the allowlist alone does NOT fix this bug. The unpin logic at L139 needs a code change to respect the allowlist, or the entire approach to pinning needs to change.

---

## Fix Options (Short-Term → Long-Term)

### Option 1 — Hardcode AFM vertical ID (minutes)

Add AFM to `isEligibleForUnpin`:

```kotlin
return primaryVerticalIds?.isNotEmpty() == true
    && !primaryVerticalIds.contains(RESTAURANT_VERTICAL_ID)
    && !primaryVerticalIds.contains(AFFORDABLE_MEALS_VERTICAL_ID)
```

Fixes AFM. Breaks again for the next Rx-like vertical with a non-1 ID.

### Option 2 — Allowlist protects against unpin (hours)

Wire the existing `boostByPositionAllowList` into the unpin filter:

```kotlin
val eligiblePairs = toBoostPairs.filter { pair ->
    isEligibleForUnpin(pair.second) && pair.second.id() !in boostByPositionAllowList
}
```

Logic becomes: "unpin NV carousels UNLESS they were explicitly configured for boosting." Fixes AFM and any future carousel in the allowlist. No new config needed — just wires up what's already there.

### Option 3 — Runtime "restaurant-equivalent" vertical IDs (days)

Replace the hardcoded `RESTAURANT_VERTICAL_ID` check with a runtime-configured set:

```kotlin
val restaurantVerticalIds = PersonalizationRuntimeUtil.restaurantEquivalentVerticalIds()
// default: [1, 100322, ...]

return primaryVerticalIds?.isNotEmpty() == true
    && !primaryVerticalIds.any { it in restaurantVerticalIds }
```

New vertical that's functionally Rx = config change, not code change. Requires a new runtime value.

### Option 4 — Allowlist cleanup + Option 2 (1-2 weeks, cross-team)

1. Parul provides SOT of all current pinning/boosting needs (non-blender only)
2. Clean stale entries from `boost_by_position_carousel_id_allow_list` runtime config
3. Apply Option 2's code change so the cleaned allowlist is the single source of truth for unpin protection

Fixes the class of problem. Requires cross-team coordination.

### Option 5 — Final Overwrite Step (RFC timeline)

Yu's north-star vision. See next section.

**Recommendation:** Option 2 now, Option 4 in parallel (Parul's audit is the bottleneck), Option 5 comes with the RFC.

---

## North Star: Yu's "Final Overwrite" Vision

Yu proposed rethinking boost/pin entirely — not as an input to ML ranking, but as a **final override after it**.

### Today's flow

```
Allowlist gates boost eligibility
  → Hinter assigns positions
    → NV unpin removes positions based on vertical ID
      → ML ranks everything
```

Pinning and unpinning happen **before** ML ranking. The unpin logic tries to decide which carousels ML should freely rank vs. which should keep their positions. This is where the AFM bug lives — the decision is based on a flawed vertical ID proxy.

### Yu's target flow

```
ML ranks everything freely (including NV carousels)
  → Final Overwrite pins allowlisted carousels to configured positions
```

There is no unpin step. Everything gets ML-ranked. Then a clean, explicit allowlist forcibly places specific carousels at their configured positions as the **last step** — a deliberate "Tony button" override. The AFM bug category disappears because the question flips from "who should we unpin?" to "who should we override to a fixed position?"

### How the Ranking Pipeline RFC Enables This

The [Ranking Pipeline RFC](./RFC.md) creates exactly the scaffold this vision needs:

**Inner ranking pipeline (RFC Milestones 1-4):** Wraps the existing ranking engine in a step-based pipeline with a generic `PipelineStep<C, S>` interface. Position boosting becomes an addressable step (`PositionBoostingSubStep` at Milestone 5) rather than logic buried inside a 258-line method.

**Post-processor pipeline (future RFC):** Decomposes `reOrderGlobalEntitiesV2()` into 9 explicit steps using the same `PipelineStep` interface. Yu's final overwrite becomes a discrete step:

```
Steps 1-7:  ML ranking, placement blending, dedup, sort order adjustments
Step 8:     FinalOverwriteStep — pins allowlisted carousels to configured positions
Step 9:     Gap rules + final ordering
```

`FinalOverwriteStep` is simple:
- Reads the cleaned allowlist (from Option 4)
- For each carousel in the allowlist with a configured position, overwrites its sort order
- Everything else keeps the ML-assigned position
- Controlled by `isApplicable()` — trivially DV-gatable

This is a clean separation of concerns:
- **ML ranking** decides the optimal order based on predictions
- **Final overwrite** applies business-mandated position locks on top
- Neither system knows about or interferes with the other

### Migration path

| Phase | What | Unpin bug status |
|---|---|---|
| **Now** | Option 2 — allowlist protects against unpin | Fixed for allowlisted carousels |
| **Option 4** | Clean up allowlist, make it the SOT | Allowlist is accurate and trusted |
| **RFC M1-4** | Inner ranking pipeline ships | No change to unpin — same code, new structure |
| **RFC M5** | Sub-step decomposition — `PositionBoostingSubStep` extracted | Unpin logic isolated, testable, independently configurable |
| **Post-processor RFC** | `FinalOverwriteStep` replaces pre-ranking pin/unpin entirely | Unpin logic removed. Pinning moves to post-ranking override. Bug category eliminated. |

---

## Key Files

| File | What |
|---|---|
| `Boosting.kt` L69-148 | `boosted()` — allowlist check (L90-98), hint generation (L110-129), NV unpin (L131-148) |
| `Boosting.kt` L693-714 | `isEligibleForUnpin()` — the vertical ID check that catches AFM |
| `VerticalIds.kt` | `RESTAURANT_VERTICAL_ID = 1`, `AFFORDABLE_MEALS_VERTICAL_ID = 100322` |
| `PersonalizationRuntimeUtil.kt` L236-242 | `boostByPositionCarouselIdAllowList()` — runtime config reader |
| `P13nExperimentManager.kt` L363-368 | `enableHomepageVerticalBlending()` — DV gate for the unpin logic |

---

## Open Questions

- [ ] Parul to provide SOT of all current pinning/boosting needs (non-blender-related only)
- [ ] Can AFM change their `primaryVerticalId`? (Michael investigating)
- [ ] Is the Sigma dashboard data correct? Pin counts looked off to Parul
- [ ] What is the right short-term fix? (recommend Option 2)
- [ ] How does gen-ai carousel boosting interact with this? (Parul confirmed gen-ai is still boosted)
