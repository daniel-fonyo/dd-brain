# E2E Sandbox Test Plan — AFM Unpin Exemption

## Strategy

We instrument the code with temporary debug logs at key decision points, deploy to sandbox, browse the homepage, and read pod logs to get semantic feedback on exactly what the code sees for each carousel. This tells us *why* a carousel is pinned or unpinned — not just that it moved.

**UI scope**: We ONLY care about two carousels visually — "Shop the Meal Box" (item) and "Best $12 meals in Miami" (store). No need to click into carousels or interact beyond scrolling to confirm their position on the homepage. All other validation is via debug logs.

## Phase 0: Debug Instrumentation

Add temporary `logger.info` calls to `Boosting.kt` that emit structured data we can grep in pod logs. **Never commit these — local only, synced via devbox.**

### Debug Log Points

All logs use a `[AFM_DEBUG]` prefix for easy grep filtering.

#### Log Point 1: DV state (in `boosted()`, after line 139)
```kotlin
logger.info("[AFM_DEBUG] enable_afm_unpin_exemption DV=$enableAfmExemption")
```

#### Log Point 2: Each carousel's eligibility decision (in `isEligibleForUnpin()`)
```kotlin
private fun isEligibleForUnpin(candidate: BoostingCandidate, enableAfmExemption: Boolean): Boolean {
    val firstStore = when (candidate) {
        is BoostingCandidate.StoreCarousel -> candidate.carousel.stores.firstOrNull()
        is BoostingCandidate.ItemCarousel -> candidate.carousel.stores.firstOrNull()
        else -> null
    } ?: return false

    val verticals = firstStore.primaryVerticalIds
    val isNv = !verticals.isNullOrEmpty() && !verticals.contains(RESTAURANT_VERTICAL_ID)
    val isAfm = enableAfmExemption && isAfmCarousel(firstStore)
    val eligible = isNv && !isAfm

    // --- DEBUG ---
    val carouselId = when (candidate) {
        is BoostingCandidate.StoreCarousel -> candidate.carousel.id
        is BoostingCandidate.ItemCarousel -> candidate.carousel.id
        else -> "unknown"
    }
    logger.info("[AFM_DEBUG] isEligibleForUnpin: carouselId=$carouselId verticals=$verticals isNV=$isNv isAfmCarousel=$isAfm eligible=$eligible")
    // --- END DEBUG ---

    return eligible
}
```

#### Log Point 3: AFM detection detail (in `isAfmCarousel()`)
```kotlin
private fun isAfmCarousel(firstStore: StoreEntity): Boolean {
    val hasAfmVertical = firstStore.primaryVerticalIds?.contains(AFFORDABLE_MEALS_VERTICAL_ID) == true
    val campaignTags = firstStore.feedCampaigns.map { it.campaignTags }
    val hasAfmCampaign = firstStore.feedCampaigns.any { isAffordableMealsMealBoxCampaign(it) }

    logger.info("[AFM_DEBUG] isAfmCarousel: storeId=${firstStore.id} hasAfmVertical=$hasAfmVertical campaignTags=$campaignTags hasAfmCampaign=$hasAfmCampaign")

    if (!hasAfmVertical) return false
    return hasAfmCampaign
}
```

#### Log Point 4: Final carousel ordering (after unpinning loop, ~line 150)
```kotlin
logger.info("[AFM_DEBUG] boosted_order=${toBoostPairs.map { it.second.let { c -> when(c) { is BoostingCandidate.StoreCarousel -> c.carousel.id; is BoostingCandidate.ItemCarousel -> c.carousel.id; else -> "other" } } }}")
logger.info("[AFM_DEBUG] unboosted_order=${unBoostPairs.map { it.second.let { c -> when(c) { is BoostingCandidate.StoreCarousel -> c.carousel.id; is BoostingCandidate.ItemCarousel -> c.carousel.id; else -> "other" } } }}")
```

### Grep Command for Logs
```bash
kubectl logs <pod> -f | grep "\[AFM_DEBUG\]"
```

## Phase 1: Sandbox Setup

### Step 1: Create feed-service worktree from PR branch
```bash
cd /Users/daniel.fonyo/Projects/feed-service
git fetch origin pull/62270/head:pr-62270
git worktree add .claude/worktrees/afm-unpin-test pr-62270
```

### Step 2: Apply local-only changes (in worktree)
1. **Consumer ID hardcode** — `HomepageRequestToContext.kt`: `return 757606047L`
2. **Debug logs** — Add all 4 log points from Phase 0 to `Boosting.kt`
3. **DO NOT** apply Iguazu bypass unless Snowflake validation is needed

### Step 3: Sync to sandbox
```bash
cd /Users/daniel.fonyo/Projects/feed-service/.claude/worktrees/afm-unpin-test
devbox run web-group1-remote
```

### Step 4: Verify sandbox is serving PR code
- Tail pod logs, look for `[AFM_DEBUG]` lines on any homepage request
- If no debug lines appear, re-sync and restart pod

---

## Phase 2: E2E Test Execution

Each test follows a structured template: **Setup → What → How → Expected → Pass Criteria → Artifacts → Results**.

---

### Test 1: Baseline — DV OFF (current prod behavior)

#### Setup
| Field | Value |
|-------|-------|
| DV `enable_afm_unpin_exemption` | `false` (default) |
| Address | 1111 Brickell Ave, Miami |
| Consumer ID | `757606047L` (hardcoded) |
| PR branch | `pr-62270` with debug instrumentation |
| Sandbox URL | `https://www.doordashtest.com/` |

#### What We're Testing
The **broken state** / current prod behavior. With the DV off, the AFM exemption is inactive. AFM carousels are treated like any other NV carousel and get unpinned by the vertical blending logic. They lose their configured sort order and get re-ranked by ML. This is the 11% adoption drop scenario.

#### How We're Testing It
1. Open `https://www.doordashtest.com/` → login with test creds
2. Set address **directly** to **1111 Brickell Ave, Miami** — skip any other navigation, go straight to this address
3. Once homepage loads, scroll to find **"Shop the Meal Box"** and **"Best $12 meals in Miami"** — only note their positions, do NOT click into any carousels
4. Take full-page screenshot and scroll recording
5. Capture pod logs: `kubectl logs <pod> -f | grep "\[AFM_DEBUG\]"`

#### Expected Debug Output
```
[AFM_DEBUG] enable_afm_unpin_exemption DV=false
[AFM_DEBUG] isEligibleForUnpin: carouselId=<afm-store> verticals=[100322] isNV=true isAfmCarousel=false eligible=true
[AFM_DEBUG] isEligibleForUnpin: carouselId=<afm-item> verticals=[100322] isNV=true isAfmCarousel=false eligible=true
[AFM_DEBUG] boosted_order=[<non-afm carousels>]
[AFM_DEBUG] unboosted_order=[<afm-store>, <afm-item>, ...]
```

#### Expected UI
- AFM carousels are **unpinned** — re-ranked by ML model
- "Shop the Meal Box" (item carousel) may still appear high due to ML scoring (user-dependent)
- "Best $12 meals in Miami" (store carousel) at ML-ranked position, NOT at configured sort order

#### Pass Criteria
- [ ] Debug logs show `DV=false`
- [ ] AFM carousels show `isAfmCarousel=false` and `eligible=true`
- [ ] AFM carousel IDs appear in `unboosted_order` list
- [ ] Screenshot captured showing positions of both AFM carousels

#### Artifacts to Capture
| Artifact | Format | Storage Location | Description |
|----------|--------|-----------------|-------------|
| Homepage screenshot (full page) | PNG | `results/t1-homepage-full.png` | Full-page screenshot showing all visible carousels with AFM positions annotated |
| Homepage scroll recording | Video/GIF | `results/t1-homepage-scroll.mp4` | Screen recording scrolling top-to-bottom, pausing on each AFM carousel |
| Debug log snippet | Text | `results/t1-debug-logs.txt` | Complete `[AFM_DEBUG]` output from pod logs for this homepage request |
| Carousel position map | Markdown table | Below in Results | Ordered list: position → carousel ID → display name → boosted/unboosted |
| Carousel ID → name mapping | Markdown table | Below in Results | Maps internal carousel IDs to visible carousel names (reused across tests) |

#### Results
> *Fill in after execution*

**Date/Time**: <!-- e.g., 2026-03-27 10:15 PST -->

**Screenshots**:
<!-- Link or embed: results/t1-homepage-full.png -->

**Scroll Recording**:
<!-- Link: results/t1-homepage-scroll.mp4 -->

**Debug Log Snippet** (copy full `[AFM_DEBUG]` output):
```
<!-- Paste here -->
```

**Carousel ID → Name Mapping** (establish once, reuse in T2/T3):
| Carousel ID | Display Name | Type |
|-------------|-------------|------|
| | Shop the Meal Box | Item |
| | Best $12 meals in Miami | Store |
| | | |

**Carousel Position Map**:
| Position | Carousel ID | Display Name | Boost Status (from logs) |
|----------|-------------|-------------|--------------------------|
| 1 | | | |
| 2 | | | |
| 3 | | | |
| ... | | | |

**Observations**:
<!-- Any unexpected behavior, notes on ML re-ranking, missing carousels, etc. -->

---

### Test 2: DV ON — AFM carousels should stay pinned

#### Setup
| Field | Value |
|-------|-------|
| DV `enable_afm_unpin_exemption` | `true` |
| Address | 1111 Brickell Ave, Miami (same session as T1) |
| Consumer ID | `757606047L` (hardcoded) |
| PR branch | `pr-62270` with debug instrumentation |
| Sandbox URL | `https://www.doordashtest.com/` |
| DV change method | Toggle in DV config, then hard-refresh (Cmd+Shift+R) |

#### What We're Testing
The **fix in action**. With the DV on, AFM carousels that have both the AFM vertical (`100322`) AND an active meal-box campaign should be exempt from unpinning. They retain their configured sort order. This is the target production state.

#### How We're Testing It
1. Toggle `enable_afm_unpin_exemption` DV to `true`
2. Hard-refresh homepage (Cmd+Shift+R) — stay on the same Miami address, don't re-navigate
3. Scroll to find **"Shop the Meal Box"** and **"Best $12 meals in Miami"** — confirm they are at their pinned positions, do NOT click into any carousels
4. Compare carousel positions to Test 1 — expect a visible shift upward
5. Take full-page screenshot, scroll recording, and side-by-side with T1
6. Capture pod logs: `kubectl logs <pod> -f | grep "\[AFM_DEBUG\]"`

#### Expected Debug Output
```
[AFM_DEBUG] enable_afm_unpin_exemption DV=true
[AFM_DEBUG] isAfmCarousel: storeId=<id> hasAfmVertical=true campaignTags=[[AFFORDABLE_MEALS_MEAL_BOX_CAMPAIGN, ...]] hasAfmCampaign=true
[AFM_DEBUG] isEligibleForUnpin: carouselId=<afm-store> verticals=[100322] isNV=true isAfmCarousel=true eligible=false
[AFM_DEBUG] isEligibleForUnpin: carouselId=<afm-item> verticals=[100322] isNV=true isAfmCarousel=true eligible=false
[AFM_DEBUG] boosted_order=[..., <afm-store>, <afm-item>, ...]
[AFM_DEBUG] unboosted_order=[<non-afm NV carousels only>]
```

#### Expected UI
- **"Shop the Meal Box"** (item carousel) pinned near **top** of homepage
- **"Best $12 meals in Miami"** (store carousel) at its **configured sort order** position
- Clear visual difference from Test 1 — both AFM carousels should be higher / in expected slots
- Non-AFM NV carousels (Grocery, Convenience) remain at ML-ranked positions

#### Pass Criteria
- [ ] Debug logs show `DV=true`
- [ ] AFM carousels show `isAfmCarousel=true` and `eligible=false`
- [ ] `isAfmCarousel` log shows `hasAfmVertical=true` and `hasAfmCampaign=true` with actual campaign tags
- [ ] AFM carousel IDs appear in `boosted_order` list (NOT in `unboosted_order`)
- [ ] "Shop the Meal Box" visible near top of homepage
- [ ] "Best $12 meals in Miami" visible at expected pinned position
- [ ] Visual position difference from Test 1 is evident for both carousels
- [ ] Other NV carousels still show `eligible=true` in logs

#### Artifacts to Capture
| Artifact | Format | Storage Location | Description |
|----------|--------|-----------------|-------------|
| Homepage screenshot (full page) | PNG | `results/t2-homepage-full.png` | Full-page screenshot with AFM carousel positions annotated |
| Homepage scroll recording | Video/GIF | `results/t2-homepage-scroll.mp4` | Screen recording scrolling top-to-bottom, pausing on each AFM carousel |
| Debug log snippet | Text | `results/t2-debug-logs.txt` | Complete `[AFM_DEBUG]` output for this homepage request |
| Carousel position map | Markdown table | Below in Results | Same format as T1 for direct comparison |
| T1 vs T2 side-by-side | Image | `results/t1-vs-t2-comparison.png` | Annotated comparison highlighting position shift of both AFM carousels |
| Position delta table | Markdown table | Below in Results | Shows T1 position → T2 position → delta for each AFM carousel |

#### Results
> *Fill in after execution*

**Date/Time**: <!-- e.g., 2026-03-27 10:30 PST -->

**Screenshots**:
<!-- Link or embed: results/t2-homepage-full.png -->

**Scroll Recording**:
<!-- Link: results/t2-homepage-scroll.mp4 -->

**T1 vs T2 Side-by-Side**:
<!-- Link: results/t1-vs-t2-comparison.png — annotated with arrows showing position changes -->

**Debug Log Snippet** (copy full `[AFM_DEBUG]` output):
```
<!-- Paste here -->
```

**Carousel Position Map**:
| Position | Carousel ID | Display Name | Boost Status (from logs) |
|----------|-------------|-------------|--------------------------|
| 1 | | | |
| 2 | | | |
| 3 | | | |
| ... | | | |

**Position Delta — T1 vs T2**:
| Carousel | T1 Position | T2 Position | Delta | Expected? |
|----------|------------|------------|-------|-----------|
| Shop the Meal Box (item) | | | | |
| Best $12 meals in Miami (store) | | | | |

**Campaign Tags Observed** (from `isAfmCarousel` log):
```
<!-- Paste the campaignTags value from the log — confirms real campaign data -->
```

**Observations**:
<!-- Notes on position shift magnitude, campaign tags content, any surprises -->

---

### Test 3: DV ON — verify non-AFM NV carousels still unpinned (regression check)

#### Setup
| Field | Value |
|-------|-------|
| DV `enable_afm_unpin_exemption` | `true` (same as T2) |
| Address | 1111 Brickell Ave, Miami (same session) |
| Consumer ID | `757606047L` (hardcoded) |
| Note | Reuses the same homepage load and pod logs from Test 2 — just analyzes different carousels |

#### What We're Testing
**No regression** for non-AFM NV carousels. The fix must be scoped: ONLY AFM carousels with active campaigns are exempt. Grocery, Convenience, Pets, and any other NV carousels must still be unpinned and re-ranked by ML. This ensures we haven't accidentally pinned everything.

#### How We're Testing It
1. Use the same pod logs captured in Test 2
2. Filter debug logs for carousels with vertical IDs other than `1` (restaurant) and `100322` (AFM)
3. Verify those carousels show `eligible=true` and appear in `unboosted_order`
4. No additional UI interaction needed — this is a log-only verification

#### Expected Debug Output
```
[AFM_DEBUG] isEligibleForUnpin: carouselId=<grocery-carousel> verticals=[<nv-id>] isNV=true isAfmCarousel=false eligible=true
[AFM_DEBUG] isEligibleForUnpin: carouselId=<convenience-carousel> verticals=[<nv-id>] isNV=true isAfmCarousel=false eligible=true
```

#### Expected UI
- No UI check needed — log-only verification
- If visible, non-AFM NV carousels should be at ML-ranked positions (no change from T1)

#### Pass Criteria
- [ ] At least one non-AFM NV carousel found in logs
- [ ] All non-AFM NV carousels show `isAfmCarousel=false` and `eligible=true`
- [ ] Non-AFM NV carousel IDs appear in `unboosted_order` list
- [ ] Restaurant carousels (vertical `1`) do NOT appear in eligibility logs (excluded before this check)

#### Artifacts to Capture
| Artifact | Format | Storage Location | Description |
|----------|--------|-----------------|-------------|
| Debug log snippet (non-AFM only) | Text | `results/t3-debug-logs-non-afm.txt` | Filtered `[AFM_DEBUG]` lines for non-AFM NV carousels only |
| Full boosted/unboosted lists | Text | `results/t3-boost-lists.txt` | Log Point 4 output showing final carousel ordering |
| Non-AFM NV carousel inventory | Markdown table | Below in Results | All non-AFM NV carousels discovered, their vertical IDs, and status |

#### Results
> *Fill in after execution*

**Date/Time**: <!-- e.g., 2026-03-27 10:35 PST -->

**Debug Log Snippet (non-AFM NV carousels only)**:
```
<!-- Paste filtered lines here -->
```

**Boosted/Unboosted Lists (Log Point 4)**:
```
<!-- Paste boosted_order and unboosted_order lines here -->
```

**Non-AFM NV Carousel Inventory**:
| Carousel ID | Vertical IDs | Display Name (if visible) | `eligible` | In `unboosted_order`? |
|-------------|-------------|--------------------------|------------|----------------------|
| | | | | |

**Observations**:
<!-- Which NV verticals were present? Any carousels with unexpected behavior? -->

---

## Rubric — Pass/Fail Summary

| # | Test | Key Assertion | Pass |
|---|------|--------------|------|
| 1 | T1: DV OFF | Debug logs confirm `DV=false` and `isAfmCarousel=false` | [ ] |
| 2 | T1: DV OFF | AFM carousels show `eligible=true` and appear in `unboosted_order` | [ ] |
| 3 | T2: DV ON | Debug logs confirm `DV=true` and `isAfmCarousel=true` | [ ] |
| 4 | T2: DV ON | AFM carousels show `eligible=false` and appear in `boosted_order` | [ ] |
| 5 | T2: DV ON | `isAfmCarousel` log confirms `hasAfmVertical=true` + `hasAfmCampaign=true` | [ ] |
| 6 | T2: DV ON | "Shop the Meal Box" (item) visible near top of homepage | [ ] |
| 7 | T2: DV ON | "Best $12 meals in Miami" (store) visible at expected pinned position | [ ] |
| 8 | T2 vs T1 | Clear position difference in screenshots/recordings for both AFM carousels | [ ] |
| 9 | T3: Regression | Non-AFM NV carousels still `eligible=true` and in `unboosted_order` | [ ] |
| 10 | All | All debug log snippets captured and pasted into results sections | [ ] |
| 11 | All | Screenshots and scroll recordings captured for T1 and T2 | [ ] |
| 12 | All | Carousel ID → display name mapping established | [ ] |

## Master Artifact Checklist

| Artifact | T1 | T2 | T3 |
|----------|----|----|-----|
| Full-page homepage screenshot | [ ] | [ ] | N/A |
| Homepage scroll recording (video/GIF) | [ ] | [ ] | N/A |
| Complete `[AFM_DEBUG]` log snippet | [ ] | [ ] | [ ] |
| Carousel position map (table) | [ ] | [ ] | [ ] |
| Carousel ID → display name mapping | [ ] | [ ] | — |
| T1-vs-T2 side-by-side comparison | — | [ ] | — |
| Position delta table (T1 vs T2) | — | [ ] | — |
| Campaign tags observed | — | [ ] | — |
| Non-AFM NV carousel inventory | — | — | [ ] |
| Boosted/unboosted list dump | — | — | [ ] |

## Phase 3: Cleanup

1. Remove all `[AFM_DEBUG]` log lines from `Boosting.kt`
2. Revert consumer ID hardcode in `HomepageRequestToContext.kt`
3. Remove feed-service worktree
4. Fill in all Results sections with collected artifacts
5. Mark rubric pass/fail
6. Commit final results to brain
