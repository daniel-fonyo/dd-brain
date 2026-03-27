# E2E Sandbox Test Plan — AFM Unpin Exemption

## Strategy

We instrument the code with temporary debug logs at key decision points, deploy to sandbox, browse the homepage, and read pod logs to get semantic feedback on exactly what the code sees for each carousel. This tells us *why* a carousel is pinned or unpinned — not just that it moved.

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

## Phase 2: E2E Test Execution

### Test 1: Baseline — DV OFF (current prod behavior)

**Setup**: `enable_afm_unpin_exemption` = `false` (default)

**Steps**:
1. Open `https://www.doordashtest.com/` → login with test creds
2. Set address to **1111 Brickell Ave, Miami**
3. Navigate to homepage
4. Screenshot the homepage carousel order
5. Check pod logs for `[AFM_DEBUG]` output

**Expected debug output**:
```
[AFM_DEBUG] enable_afm_unpin_exemption DV=false
[AFM_DEBUG] isEligibleForUnpin: carouselId=<afm-store> verticals=[100322] isNV=true isAfmCarousel=false eligible=true
[AFM_DEBUG] isEligibleForUnpin: carouselId=<afm-item> verticals=[100322] isNV=true isAfmCarousel=false eligible=true
```

**Expected UI**: AFM carousels are **unpinned** — "Shop the Meal Box" may still appear high (ML re-ranking) but "Best $12 meals" should be at ML-ranked position, not pinned sort order.

**Pass criteria**:
- [ ] Debug logs show `DV=false` and `isAfmCarousel=false` for AFM carousels
- [ ] Both AFM carousels show `eligible=true` (will be unpinned)
- [ ] Screenshot captured showing carousel positions

---

### Test 2: DV ON — AFM carousels should stay pinned

**Setup**: Set `enable_afm_unpin_exemption` = `true` in DV config

**Steps**:
1. Toggle DV to `true`
2. Hard-refresh homepage (or wait for DV cache to expire)
3. Screenshot the homepage carousel order
4. Check pod logs

**Expected debug output**:
```
[AFM_DEBUG] enable_afm_unpin_exemption DV=true
[AFM_DEBUG] isAfmCarousel: storeId=<id> hasAfmVertical=true campaignTags=[[AFFORDABLE_MEALS_MEAL_BOX_CAMPAIGN, ...]] hasAfmCampaign=true
[AFM_DEBUG] isEligibleForUnpin: carouselId=<afm-store> verticals=[100322] isNV=true isAfmCarousel=true eligible=false
[AFM_DEBUG] isEligibleForUnpin: carouselId=<afm-item> verticals=[100322] isNV=true isAfmCarousel=true eligible=false
```

**Expected UI**: AFM carousels are **pinned** — "Shop the Meal Box" at top, "Best $12 meals in Miami" at its configured sort order position.

**Pass criteria**:
- [ ] Debug logs show `DV=true` and `isAfmCarousel=true` for AFM carousels
- [ ] Both AFM carousels show `eligible=false` (won't be unpinned)
- [ ] AFM carousels appear at pinned positions vs Test 1
- [ ] Other NV carousels (if visible) still show `eligible=true`

---

### Test 3: DV ON, but verify non-AFM NV carousels still unpinned

**Setup**: DV still `true`

**Steps**:
1. Same homepage from Test 2
2. Look at debug logs for non-AFM NV carousels (e.g., Grocery, Convenience — vertical IDs other than 1 and 100322)

**Expected debug output**:
```
[AFM_DEBUG] isEligibleForUnpin: carouselId=<grocery> verticals=[<nv-id>] isNV=true isAfmCarousel=false eligible=true
```

**Pass criteria**:
- [ ] Non-AFM NV carousels show `eligible=true` in logs
- [ ] They are in the unboosted list, not the boosted list

---

## Rubric — Pass/Fail Summary

| Test | Key Assertion | Pass |
|------|--------------|------|
| T1: DV OFF | AFM carousels show `eligible=true`, get unpinned | [ ] |
| T2: DV ON | AFM carousels show `eligible=false`, stay pinned at configured position | [ ] |
| T2: DV ON | "Shop the Meal Box" (item) visible near top of homepage | [ ] |
| T2: DV ON | "Best $12 meals in Miami" (store) visible at expected position | [ ] |
| T3: DV ON | Non-AFM NV carousels still `eligible=true`, still unpinned | [ ] |
| Visual | Clear position difference between T1 and T2 screenshots for AFM carousels | [ ] |

## Artifacts to Collect

For each test:
1. **Screenshot** of homepage showing carousel ordering
2. **Pod log snippet** — all `[AFM_DEBUG]` lines from that request
3. **Carousel ID mapping** — which carousel ID maps to which visible carousel name

## Phase 3: Cleanup

1. Remove all `[AFM_DEBUG]` log lines from `Boosting.kt`
2. Revert consumer ID hardcode in `HomepageRequestToContext.kt`
3. Remove feed-service worktree
4. Update this doc with test results and screenshots
