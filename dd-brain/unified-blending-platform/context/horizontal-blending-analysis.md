# Horizontal Blending Analysis

> Ads insertion into organic store lists — the ICP (Inventory Control Plane) horizontal blending system.

---

## Two Meanings of "Horizontal Blending"

When someone says "horizontal blending," they might mean either:

| Meaning | What it is | Where in pipeline | UBP phase |
|---|---|---|---|
| **Organic horizontal ranking** | Ranking stores within a carousel by ML score | Layer 3 (before post-processing) | Phase 1-2 |
| **Ads horizontal blending** | Inserting ad store cards into ranked organic lists | Parallel to post-processing | Phase 3 |

This document covers **meaning 2**: the ICP ads blending system.

---

## Where It Fits in the Pipeline

```
retrieval → grouping → horizontal ranking → decoration → layout
                                                              │
                                                    ┌─────────┴──────────┐
                                                    │                    │
                                             postProcessing        HomepageAdsBlender
                                           (vertical ranking)      (ads horizontal blend)
                                                    │                    │
                                                    └─────────┬──────────┘
                                                              │
                                                        SERIALIZE → CLIENT
```

`HomepageAdsBlender.blend()` runs **in parallel** with post-processing in the pipeline DAG. Both receive the organically-ranked content as input; their outputs are merged before serialization.

---

## The Four Blending Targets

`HomepageAdsBlender.blend()` spawns a Workflow with 5 jobs (some sequential via `dependsOn`):

```
┌─────────────────────────────────────────────────────────┐
│ smartSuggestionsAdsBlendingJob                          │
│   surface: IcpSurface.SL_SMART_SUGGESTIONS              │
│   blend ads into SmartSuggestionCollection              │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ storeCarouselAdsBlenderJob                              │
│   surface: IcpSurface.SL_CAROUSEL                       │
│   adsBlendingDataPlane.twoDimensionalSLBlend()          │
│   = carousel-level placement + slot-level placement     │
└──────────────────────────────┬──────────────────────────┘
                               │ (dependsOn)
┌──────────────────────────────▼──────────────────────────┐
│ storeListAdsBlenderJob                                  │
│   surface: IcpSurface.SL_HOME_FEED                      │
│   blend ads into main home feed store list              │
│   dedupes with ads already placed in carousels above    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ padAdsBlenderJob                                        │
│   PAD (Personalized Ad Display) collections             │
└──────────────────────────────┬──────────────────────────┘
                               │ (dependsOn)
┌──────────────────────────────▼──────────────────────────┐
│ immersiveContainerAdsBlenderJob                         │
│   Immersive container (spotlight banner) blending       │
└─────────────────────────────────────────────────────────┘
```

All 5 jobs run concurrently where possible; failures are non-critical (falls back to organic-only).

---

## Two-Dimensional Blend (Carousels)

For store carousels, `twoDimensionalSLBlend()` does blending in two dimensions:

1. **Vertical (carousel-level)**: Which carousels receive ad slots at all? (Not all carousels are ad-eligible)
2. **Horizontal (slot-level)**: Within an ad-eligible carousel, which slots get ads vs. organic stores?

The horizontal slot blending is what this document primarily covers.

---

## Three Blending Algorithms

Selected via `HorizontalBlenderConfig.blendingAlgorithm`:

### 1. GenericHorizontalBlenderV2 (primary)

**Strategy**: Cluster-based interleaving. For each ad-organic cluster in sequence, generate all possible ad sequences that satisfy constraints, pick the first one without ghost ads.

```
Config: configuredClusters=[(A1,O6),(A2,O6)], repeatCluster=(A1,O4)

Position pattern:
  [A] [O] [O] [O] [O] [O] [O]   ← first configured cluster: 1 ad, 6 organic
  [A] [A] [O] [O] [O] [O] [O] [O]   ← second: 2 ads, 6 organic
  [A] [O] [O] [O] [O]   ← repeat forever: 1 ad, 4 organic
  [A] [O] [O] [O] [O]
  ...
```

**Constraints per ad slot**:
1. Universal quality floor (ad score ≥ threshold)
2. Position quality floor (per-slot override, e.g., stricter floor for slot 0)
3. Dedupe radius (ad cannot be within `dedupeRadius` positions of same-business organic)
4. Lift experiment fairness (only 1 ad per lift campaign per blend)
5. Must not be a ghost ad (ghost ads tracked but not shown)

**Ghost ads**: If the best eligible sequence includes a ghost ad (control group of a lift experiment), that ghost ad is recorded on the adjacent candidate for iROAS measurement, and blending continues looking for real ad sequences.

### 2. DedupeLastHorizontalBlender

**Strategy**: Aggressive deduplication. First occurrence of a store ID wins (whether ad or organic). All subsequent duplicates are silently dropped.

Key difference from GenericV2:
- No dedupeRadius constraint — deduplication is global across the entire list
- Supports **pagination** via `AdsBlendingCursor` — cursor carries `clusterOffset`, `dedupeIds`, `globalItemCount` across page requests

Used for surfaces where carousel-level dedup radius doesn't apply.

### 3. SPLegacyHorizontalBlender

Legacy algorithm kept for backward compatibility. Not used for new surfaces.

---

## HorizontalBlenderConfig Contract

```json
{
  "blending_algorithm": "generic_1D_blending",
  "start_index": 0,
  "configured_slot_sizes": [
    { "ad_cluster_size": 1, "organic_cluster_size": 6 },
    { "ad_cluster_size": 2, "organic_cluster_size": 6 }
  ],
  "repeated_slot_size": { "ad_cluster_size": 1, "organic_cluster_size": 4 },
  "num_ads_limit": 5,
  "min_organic_limit": 3,
  "universal_quality_floor": 0.002,
  "universal_quality_percentile_floor": null,
  "dedupe_radius": 2,
  "position_quality_floors": [
    { "start_index": 0, "end_index": 3, "quality_floor": 0.05, "relevance_floor": 1.5 }
  ],
  "prevent_ad_from_blocking_all_organics": true
}
```

| Field | Meaning |
|---|---|
| `start_index` | Number of organic slots before any ad cluster begins |
| `configured_slot_sizes` | Non-repeating prefix of ad/organic clusters |
| `repeated_slot_size` | Repeating pattern after prefix |
| `num_ads_limit` | Hard cap on total ads per blend |
| `min_organic_limit` | Minimum organics needed to enable blending at all |
| `universal_quality_floor` | Minimum ad quality score (absolute) |
| `universal_quality_percentile_floor` | Percentile-based quality override (experiment-gated) |
| `dedupe_radius` | Min positions between an ad and same-business organic |
| `position_quality_floors` | Per-position stricter quality floors |
| `prevent_ad_from_blocking_all_organics` | True = never let an ad be the only thing blocking all remaining organics |

Default config: `OneDimBlending`, `maxNumAds=10`, `universalQualityFloor=0.002`, `dedupeRadius=1`, `repeatCluster=(A1, O4)`.

---

## Ghost Ads and iROAS Measurement

Lift experiments split ads into:
- **Treatment group**: sees real ads
- **Control group**: sees ghost ads (ads that meet all quality floors but are not shown)

Ghost ads are attached to adjacent candidates via `ghostAdCandidates` on the `Candidate` interface. The serializer records ghost ad metadata for iROAS (incremental Return on Ad Spend) measurement. The blending constraint is: at most 1 ghost ad per cluster (otherwise fairness between treatment and control breaks).

---

## The Root Problem: Incomparable Scores

```
Organic ranking score  = Sibyl ML predicted CTR/CVR (calibrated to order probability)
Ad quality score       = Output of ad auction (bid × predicted CTR from ads model)
```

These are on **completely different scales**. The blending system cannot answer "is this ad more valuable than the organic store it's displacing?" — it can only enforce fixed slot patterns.

Result: ads are inserted at fixed positions (slot 0 = ad, slots 1-4 = organic, ...) regardless of whether the displaced organic store had higher expected value.

**UBP Phase 3 fix**: calibration service normalizes both to `pAct × vAct` on a common scale → ads and organics compete for the same slots → highest-EV content wins regardless of type.

---

## UBP Phase Roadmap

| Phase | What | Ads Blending Impact |
|---|---|---|
| Phase 1 | Vertical ranking engine | None — ads blending unaffected |
| Phase 2 | Horizontal ranking engine | None — ads blending unaffected |
| Phase 3 | Value function + calibration | Organic ML scores and ad scores on same scale → true competition → no fixed slots |

Phase 3 is when `ADS_BLEND` becomes a first-class `HorizontalProcessor` step in the pipeline config, rather than a separate parallel job.

---

## Key Files

| File | What |
|---|---|
| `pipelines/homepage/.../HomepageAdsBlender.kt` | 5-job blending workflow; orchestrates all 4 surfaces |
| `libraries/domain-util/.../AdsBlendingDataPlane.kt` | `twoDimensionalSLBlend()`: carousel-level + slot-level dispatch |
| `libraries/domain-util/.../GenericHorizontalBlenderV2.kt` | Primary blending algorithm; `AdClusterGenerator` recursive DFS |
| `libraries/domain-util/.../DedupeLastHorizontalBlender.kt` | Aggressive dedup strategy; pagination support |
| `libraries/domain-util/.../BaseHorizontalBlender.kt` | Shared utilities: `getSortedAdsAboveQualityFloor()`, `getClusterSequence()` |
| `libraries/domain-util/.../BlenderFactory.kt` | `resolve()`: maps `blendingAlgorithm` string → blender instance |
| `libraries/domain-util/.../models/configs/HorizontalBlenderConfig.kt` | Full config contract with all fields |
| `libraries/domain-util/.../models/quality/HorizontalQualityFloorFilter.kt` | Quality floor evaluation logic |
