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

### Implication
The S1/S2/S3 heuristics appear to be the mechanism that drives volume toward high-inflation merchants, not the inflation multiplier itself. Treatment2 (pure ML ranking) shifts volume away from high-inflators.

---

## Open Questions / Debug Items
- TODO: Validate whether the stat sig T2 result holds at 2 weeks
- TODO: Check other core metrics (orders, GMV, Cx satisfaction) for T2 to confirm no negative tradeoffs
- TODO: Understand what S1/S2/S3 heuristics are doing that pushes volume toward high-inflators
