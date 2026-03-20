# Favorites Inflation Override Experiment

**DV Name:** `favorites-inflation-override-vp-2026`
**PR:** https://github.com/doordash/feed-service/pull/60043
**Branch:** `inflation_overide_reorder`
**Status:** Merged, running ~1 week (as of 2026-03-19)
**Author:** Dipali Ranjan

---

## Goal

Test removing the inflation multiplier and S1/S2/S3 heuristic reranking from the **favorites carousel** on homepage. Instead of tiger-team heuristics, rank by Homepage Store Ranker (pure ML).

## Experiment Design

| Bucket | Inflation Multiplier | S1/S2/S3 Heuristics | Ranking Type | Purpose |
|--------|---------------------|---------------------|--------------|---------|
| **Control** | ON (`inflation_sr_multiplier_3_1`) | ON | `INFLATION_MULTIPLIER_CAROUSEL` | Current baseline with both mechanisms |
| **Treatment1** | OFF | ON | `CAROUSEL` | Isolate impact of removing inflation multipliers |
| **Treatment2** | OFF | OFF | `CAROUSEL` | Pure ML ranking, no adjustments |

### Comparisons
- **T1 vs Control** — isolates inflation multiplier impact
- **T2 vs T1** — isolates S1/S2/S3 heuristic impact
- **T2 vs Control** — combined impact of removing both

---

## Code Changes (PR #60043)

### 1. DV Infrastructure

**DiscoveryExperimentManager.kt**
- Added DV manifest: `FAVORITES_INFLATION_OVERRIDE_VP_2026("favorites-inflation-override-vp-2026")`

**DiscoveryRuntimeUtil.kt**
- Added `getInflationOverrideExperimentFeatureMapping()` reading runtime config from: `ranking/reorder_carousel_override.json`

**TigerTeamUtil.kt**
- Added `isFavoritesInflationOverrideEnabled()` — mirrors `isTigerTeamFeatureEnabled()` but reads from the inflation override runtime config
- Added `DISABLE_FAVORITES_HEURISTIC_RERANKING` to `TigerTeamFeature` enum

### 2. FavoritesCarousel.kt — Core Logic

#### A. `getRankingType()` (lines 165-198)

Two separate checks with **independent** scopes:

```kotlin
val isTigerFeatureActive = TigerTeamUtil.isTigerTeamFeatureEnabled(
    featureName = DISABLE_ORDER_AGAIN_INFLATION_FILTER,
    experimentMap = context.experimentMap,
)  // Tiger team DV — applies to ALL page types

val isInflationOverrideActive = TigerTeamUtil.isFavoritesInflationOverrideEnabled(
    featureName = DISABLE_ORDER_AGAIN_INFLATION_FILTER,
    experimentMap = context.experimentMap,
)  // Inflation override DV — scoped to HOMEPAGE only
```

**Ranking type resolution order:**
1. `isTigerFeatureActive` → `CAROUSEL` (all page types, original tiger team)
2. Shopping tab → `FAVORITES_CAROUSEL`
3. `isInflationOverrideActive` AND homepage → `CAROUSEL` (this experiment)
4. Homepage + `homepage_inflation_filter_ranker_v3 == TREATMENT` → `INFLATION_MULTIPLIER_CAROUSEL`
5. Homepage + reorder carousel enabled → `REORDER_CAROUSEL`
6. Fallback → `super.getRankingType()`

**Key design:** Treatment branches return `CAROUSEL` **before** checking other P13N experiments (e.g., `discovery_p13n_reorder_store_carousel`), preventing experiment contamination.

#### B. `rerankDecoratedEntitiesUtil()` (lines 286-321)

```kotlin
val isInflationOverrideHeuristicDisabled = TigerTeamUtil.isFavoritesInflationOverrideEnabled(
    featureName = DISABLE_FAVORITES_HEURISTIC_RERANKING,
    experimentMap = experimentMap,
)
```

S1/S2/S3 heuristics only apply when ALL of:
- `isTigerFeatureActive == true`
- `isTigerHeuristicDisabled == false`
- `isInflationOverrideHeuristicDisabled == false` ← **new check**
- `filter != null`

Treatment2 maps `DISABLE_FAVORITES_HEURISTIC_RERANKING` in the runtime config, so heuristics are skipped.

### 3. Runtime Config

**Path:** `ranking/reorder_carousel_override.json`

Expected structure:
```json
{
  "control": [],
  "treatment1": ["DISABLE_ORDER_AGAIN_INFLATION_FILTER"],
  "treatment2": ["DISABLE_ORDER_AGAIN_INFLATION_FILTER", "DISABLE_FAVORITES_HEURISTIC_RERANKING"]
}
```

---

## Key Metric: `vol_from_over_25`

**Definition:** Percentage of volume going to highest-inflation merchants (25%+ markup relative to in-store prices).

**Why it matters:** DoorDash promises Mx that lower markup → better visibility/volume. This metric ensures that commitment is upheld. Left unchecked, volume incentives based on inflation rate erode.

**Promotion:** Being promoted to CX core metric pack (requestors: Bri Lister, Luis Martinez-Moure).

---

## Current Results (~1 week)

| Metric | Control | Treatment1 | Treatment2 |
|--------|---------|------------|------------|
| **vol_from_over_25** | 0.177552 | 0.177561 | 0.176826 |
| **N (observations)** | 3,333,615 | 3,234,185 | 3,236,850 |
| **Absolute delta vs control** | — | -0.000032 [-0.000472, +0.000408] | **-0.000705** [-0.001144, -0.000265] |
| **Relative delta vs control** | — | -0.0179% [-0.2657%, +0.2299%] | **-0.3968%** [-0.6441%, -0.1495%] |
| **p-value** | — | 0.887605 | **0.001665** |
| **Positive direction** | — | 37.3347% | 37.3347% |
| **Absolute delta (alt view)** | — | -0.0067% | -0.1477% |
| **Second absolute delta** | — | -0.000012 | -0.00026 |

### Interpretation

- **Treatment1 (no inflation, keep heuristics):** Flat — no stat sig change in vol_from_over_25 (p=0.89). Removing inflation multiplier alone doesn't move high-inflation volume.
- **Treatment2 (no inflation, no heuristics):** Stat sig decrease in vol_from_over_25 (p=0.0017). Removing both mechanisms reduces volume to 25%+ inflators by ~0.4% relative. This is directionally **good** — less volume to high-markup merchants.

---

## Analysis: Why T1 Shows No Effect (p=0.89)

### Confirmed: Ranking type change IS working

Tiger team runtime config (`tiger_team/experiment_feature_mapping.json`) does **not** include `DISABLE_ORDER_AGAIN_INFLATION_FILTER` in any bucket:

```
control:              []
treatment1:           [CONTENT_SYSTEM_DAY_PART_CX_AFFINITY]
treatment2:           [... ORDER_AGAIN_SR_OVERRIDDEN]
treatment3:           [... ORDER_AGAIN_SR_OVERRIDDEN, DISABLE_THE_CAROUSEL_CAPPING]
treatment4:           []
```

So check 1 (`isTigerFeatureActive` for `DISABLE_ORDER_AGAIN_INFLATION_FILTER`) is **always false**. The ranking type path works as designed:

- Control → `INFLATION_MULTIPLIER_CAROUSEL` (inflation ON)
- T1 → `CAROUSEL` (inflation OFF)
- T2 → `CAROUSEL` (inflation OFF)

**T1 is genuinely changing the ranking type.** Yet p=0.89.

### Heuristic gate: only tiger team t2/t3

The heuristic reranking in `rerankDecoratedEntitiesUtil` checks a **different** feature:

```kotlin
val isTigerFeatureActive = TigerTeamUtil.isTigerTeamFeatureEnabled(
    featureName = TigerTeamFeature.ORDER_AGAIN_SR_OVERRIDDEN,  // different from ranking check
    experimentMap = experimentMap,
)
```

`ORDER_AGAIN_SR_OVERRIDDEN` only exists in tiger team **treatment2 and treatment3**. So:

| Tiger team bucket | Heuristics available? |
|-------------------|-----------------------|
| treatment1 | NO |
| treatment2 | YES |
| treatment3 | YES |
| treatment4 | NO |

Heuristics are OFF for ~half the tiger team population (t1/t4) and all non-tiger-team users.

### Corrected PR table

| Bucket | Inflation Multiplier | Heuristics (PR claim) | Heuristics (actual) |
|--------|---------------------|----------------------|---------------------|
| Control | ON | ON | **ON only for tiger team t2/t3** |
| T1 | OFF | ON | **ON only for tiger team t2/t3** |
| T2 | OFF | OFF | OFF |

### Hypothesis: heuristics override the ranking change

For tiger team t2/t3 users, the call stack is:
1. Store Ranker produces ranked list (with or without inflation multiplier)
2. `rerankDecoratedEntitiesUtil` runs S1/S2/S3 heuristics **after** ranking
3. Heuristics re-sort stores, potentially overriding the ranker's ordering

```
Control (t2/t3 user):
  Store Ranker (INFLATION_MULTIPLIER_CAROUSEL) → [ranked list] → S1/S2/S3 rerank → [final order]

T1 (t2/t3 user):
  Store Ranker (CAROUSEL, no inflation)        → [ranked list] → S1/S2/S3 rerank → [final order]
                                                                  ↑
                                              if heuristics dominate, both produce similar final order
```

**If heuristics are strong enough to override the input ranking**, then for t2/t3 users the inflation multiplier removal has no net effect — heuristics push high-inflators back into the same positions regardless.

For non-t2/t3 users (heuristics OFF), T1 should show the clean effect of removing the inflation multiplier. But this subgroup may be too small or the inflation multiplier may genuinely not affect `vol_from_over_25` for these users either.

**We need to trace the full call stack to confirm whether something downstream re-sorts the favorites carousel after the ranking type is applied.**

---

## Sandbox Debug Trace (2026-03-20)

### Setup

**Debug branch:** `debug/favorites-inflation-callstack` in feed-service
**Sandbox:** rd42e4ff, pod feed-service-web-group1-sandbox-rd42e4ff
**Run ID:** 2026-03-20T15-35-04-2b093b
**Consumer hardcoded:** 757606047L (inflation override bucket: treatment1)

Added `[DEBUG-FAVORITES-INFLATION]` logs at every point where favorites carousel store ordering can change:

| File | Instrumented Steps |
|------|-------------------|
| `FavoritesCarousel.kt` | `getRankingType` — logs ranking type, bucket values, DV flags |
| `FavoritesCarousel.kt` | `rerankInput/rerankOutput` — logs rerank branch, order_changed |
| `HomepageFavoritesProduct.kt` | `rerankInput/rerankOutput` — wrapper-level store IDs |
| `DefaultHomePageStoreRanker.kt` | `scoreSort`, `bizRulesSort`, `paginated` — 3-stage ranking |
| `DefaultHomePagePostProcessor.kt` | `processContent_input/finalOutput`, dedup stages |

### Results: 357 debug log lines, 12 unique consumers

| Metric | Distribution |
|--------|-------------|
| `ranking_type` | 126x CAROUSEL, 0x INFLATION_MULTIPLIER_CAROUSEL |
| `rerank_branch` | 18x no_rerank, 5x s1s2s3_heuristics |
| `tiger_team_bucket` | 32x control, 9x treatment2 |
| `inflation_override_bucket` | 33x treatment1, 8x control |
| `order_changed` (rerank) | **31x false, 0x true** |
| `order_changed_from_score` (bizRules) | 50x true, 4x false |
| Post-processor changes order | 8/19 pairs changed (~42%) |

### Key Findings

**1. Ranking type is CAROUSEL for all T1 users — experiment DV is working.**
All 18 `getRankingType` calls for inflation_override_bucket=treatment1 return `CAROUSEL`. Confirmed.

**2. Reranking NEVER changes store order.**
All 31 `rerankOutput` entries show `order_changed: false`. For tiger team control users (majority), the branch is `no_rerank` because `ORDER_AGAIN_SR_OVERRIDDEN` is false. For tiger team treatment2 users (5 cases), the branch is `s1s2s3_heuristics` but still `order_changed: false` — meaning the heuristics evaluate but produce the same ordering as the input.

**3. Business rules sort overrides score sort 93% of the time.**
50 out of 54 `storeRanker_bizRulesSort` calls show `order_changed_from_score: true`. The business rules sort runs AFTER score sort inside `DefaultHomePageStoreRanker.sortStoreEntitiesForCarousels()`.

**4. Post-processor cross-carousel dedup changes order ~42% of the time.**
8 out of 19 `processContent_input` to `processContent_finalOutput` pairs show different store order. This is due to stores being removed from favorites carousel because they appeared in earlier carousels.

### Root Cause: Multiplier Model Difference

The ranking type change controls which **multiplier model** is used in `CollectionScorer.getMultiplierModelName()`:

```kotlin
// CollectionScorer.kt:1065-1072
private fun getMultiplierModelName(...): String? {
    return if (context.rankingType == RankingType.INFLATION_MULTIPLIER_CAROUSEL) {
        INFLATION_RANKER_MODEL_NAME  // "inflation_sr_multiplier_3_1"
    } else {
        modelMap[STORE_RANKER_MULTIPLIER_SIBYL_PREDICTOR_NAME.label]
            ?.getModelNameFromExperiment(context.experimentMap)  // standard multiplier
    }
}
```

Final score formula in `SibylRegressor`:
```
score = base_score x multiplier_prediction x programmatic_boost + uncertainty + mab
```

So T1 (CAROUSEL) uses a **different multiplier model** than Control (INFLATION_MULTIPLIER_CAROUSEL). This produces different prediction scores. However, the business rules sort downstream overrides the score-based ordering 93% of the time, washing out the multiplier model difference.

### Full Call Stack (verified)

```
1. FavoritesCarousel.getRankingType()
   Control: INFLATION_MULTIPLIER_CAROUSEL
   T1/T2:  CAROUSEL

2. CollectionScorer.getMultiplierModelName()
   INFLATION_MULTIPLIER_CAROUSEL: "inflation_sr_multiplier_3_1"
   CAROUSEL: standard multiplier from experiment config
   --> Different multiplier models produce different prediction scores

3. SibylRegressor.predict()
   score = base_score x multiplier x boost + uncertainty + mab
   --> Scores ARE different between Control and T1

4. DefaultHomePageStoreRanker.sortStoreEntitiesForCarousels()
   a. Score sort (by prediction scores) — different between Control/T1
   b. Business rules sort — overrides score sort 93% of the time <-- WASH-OUT POINT
   c. Pagination

5. HomepageFavoritesProduct.rerankDecoratedEntities()
   FavoritesCarousel.rerankDecoratedEntitiesUtil()
   Tiger team control users: no_rerank (no-op)
   Tiger team t2/t3 users: s1s2s3_heuristics (but order_changed=false)
   --> Reranking never changes order

6. DefaultHomePagePostProcessor.processContent()
   a. Placement dedup
   b. Trim carousels
   c. Cross-carousel dedup — changes order ~42% of time (removes stores seen earlier)
   d. Final output
```

### Why T1 = No Effect (CONFIRMED)

The inflation multiplier model does produce different scores, but **business rules sort in step 4b is the dominant ordering signal**. It overrides score-based ordering for 93% of carousel instances. The multiplier model difference between `inflation_sr_multiplier_3_1` and the standard model is too weak relative to business rules to change the final store order meaningfully.

Additionally, the S1/S2/S3 heuristics (which T1 keeps ON) don't actually change store order in practice (`order_changed: false` in all 31 cases), so they're not masking anything — they're already a no-op for the observed traffic.

### Why T2 = Significant -0.4%

T2 disables BOTH the inflation multiplier AND the S1/S2/S3 heuristic reranking. While heuristics showed `order_changed: false` in our sandbox trace (single consumer), the key difference is that T2 also sets `DISABLE_FAVORITES_HEURISTIC_RERANKING` which fully bypasses the heuristic code path. This may interact differently at scale across the full population — tiger team t2/t3 users with diverse store mixes may see heuristic effects that our single-consumer trace didn't capture.

**Alternative T2 explanation**: The statistical significance may come from the combined removal of both mechanisms creating a measurably different carousel composition at the population level, even if individual effect sizes are small.

---

## Next Steps

- [x] ~~Trace favorites carousel call stack end-to-end~~ — **Done.** See sandbox debug trace above.
- [ ] **Segment T1 results by tiger team bucket** — compare T1 vs Control for t2/t3 users (heuristics ON) vs t1/t4 users (heuristics OFF)
- [ ] **Validate T2 at 2 weeks** — confirm stat sig holds
- [ ] **Check core metrics for T2** — orders, GMV, Cx satisfaction for negative tradeoffs
- [ ] **Investigate business rules sort dominance** — understand what `bizRulesSort` does in `DefaultHomePageStoreRanker` and whether it's intentionally overriding ML scores. If so, the multiplier model is irrelevant for favorites carousel ranking.
- [ ] **Compare multiplier model outputs** — get actual score distributions for `inflation_sr_multiplier_3_1` vs standard model to quantify the difference
- [ ] **Test with heuristic-active user** — run sandbox trace with a consumer in tiger team t2/t3 who has diverse store mix to see if heuristics actually change order for them
