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

## Bug: Tiger Team Overlap Makes T1 a No-Op

### The problem

The `getRankingType()` if/else chain fires **top to bottom, first match wins**:

```
check 1: if (isTigerFeatureActive)                          → CAROUSEL
check 2: else if (shoppingTab)                              → FAVORITES_CAROUSEL
check 3: else if (isInflationOverrideActive && homepage)    → CAROUSEL
check 4: else if (homepage && inflation_ranker_v3 == TREATMENT) → INFLATION_MULTIPLIER_CAROUSEL
check 5: else if (homepage && reorder_carousel)             → REORDER_CAROUSEL
check 6: else                                               → fallback
```

**Check 1 (tiger team) fires before check 3 (this experiment) and check 4 (inflation multiplier).**

### DV dependency chain

```
sdk-dynamic-taste-homepage-rollout
  └─ contained in treatment4?
       └─ YES → hp-tiger-team-2025 (t1-t4, all 100% → treatment)
                  └─ isTigerFeatureActive = true → check 1 fires

discovery_p13n_inflation_ranker_v2
  ├─ 90% → treatment3 → homepage_inflation_filter_ranker_v3 = TREATMENT
  └─ 10% → control    → homepage_inflation_filter_ranker_v3 = CONTROL
```

### What happens for tiger team users

If a user is in tiger team treatment, check 1 fires and returns `CAROUSEL` for **all three buckets**. Check 3 (inflation override) and check 4 (inflation multiplier) are never reached.

```
Tiger team users:
  Control:  check 1 → CAROUSEL  (inflation already OFF — tiger team turned it off)
  T1:       check 1 → CAROUSEL  (inflation already OFF — same as control)
  T2:       check 1 → CAROUSEL  (inflation already OFF — same as control)
  ↑
  All three buckets are IDENTICAL for ranking type.
```

### What happens for non-tiger-team users

Check 1 is skipped. The experiment works as designed:

```
Non-tiger-team users:
  Control:  check 4 → INFLATION_MULTIPLIER_CAROUSEL  (inflation ON)
  T1:       check 3 → CAROUSEL                       (inflation OFF)
  T2:       check 3 → CAROUSEL                       (inflation OFF)
```

### PR table vs reality

| Bucket | PR says: Inflation | Tiger team users (actual) | Non-tiger-team users (actual) |
|--------|--------------------|--------------------------|-----------------------------|
| Control | ON | **OFF** (check 1 → CAROUSEL) | ON (check 4 → IMC) |
| T1 | OFF | **OFF** (check 1 → CAROUSEL) | OFF (check 3 → CAROUSEL) |
| T2 | OFF | **OFF** (check 1 → CAROUSEL) | OFF (check 3 → CAROUSEL) |

The PR table is only accurate for non-tiger-team users. For tiger team users, **Control already has inflation OFF**, making T1 identical to Control.

### Why T2 still works

The heuristic disable (`rerankDecoratedEntitiesUtil`) is a **separate code path** that runs after ranking type is selected. It's not gated by the if/else chain above. So T2's heuristic disable applies to ALL users regardless of tiger team status:

```
Tiger team users:     Control = T1 (heuristics ON) ≠ T2 (heuristics OFF)  ← T2 differs
Non-tiger-team users: Control ≠ T1 (ranking type) ≠ T2 (ranking + heuristics)
```

### Impact on results

```
T1 overall effect ≈ (%tiger × 0) + (%non-tiger × real_inflation_effect)
                         ↑ no-op           ↑ diluted into total
                   = heavily diluted → p=0.89

T2 overall effect ≈ (%tiger × heuristic_effect) + (%non-tiger × inflation + heuristic effect)
                         ↑ real signal                    ↑ real signal
                   = signal from entire population → p=0.0017
```

### Conclusion

T1's p=0.89 does NOT mean the inflation multiplier has no effect. It means **T1 is a no-op for the tiger-team subpopulation**, which dilutes any real signal. We cannot draw conclusions about inflation multiplier impact from T1 without segmenting out tiger team users.

T2's stat sig result is driven by heuristic removal, which works across the entire population.

---

## Open Questions / Next Steps

- **Determine tiger team population overlap** — what % of this experiment's users are in tiger team? This tells us how diluted T1 is.
- **Segment T1 results** — look at T1 vs Control for non-tiger-team users only. This isolates the actual inflation multiplier effect.
- **Validate T2 at 2 weeks** — confirm stat sig holds
- **Check core metrics for T2** — orders, GMV, Cx satisfaction to confirm no negative tradeoffs
- **Understand S1/S2/S3 heuristics** — what specifically are they doing that pushes volume toward high-inflators?
