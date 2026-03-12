# Horizontal Blending Deep Dive: Ads Insertion Into Organic Carousels

> Source: `HomepageAdsBlender.kt`, `BaseHorizontalBlender.kt`, `GenericHorizontalBlenderV2.kt`,
> `DedupeLastHorizontalBlender.kt`, `HorizontalBlenderConfig.kt`, `BlenderFactory.kt`

---

## The Short Answer

Yes, horizontal blending exists — but it is a **completely different concept** from organic
horizontal ranking and must not be confused with it.

```
ORGANIC HORIZONTAL RANKING     HORIZONTAL ADS BLENDING
(restaurantStoreRankingJob)     (HomepageAdsBlender, separate job)
────────────────────────────    ─────────────────────────────────
Ranks stores by ML score        Takes the already-ranked organic list
within a carousel.              and INSERTS ad store cards into it
                                at specific slot positions.

Input: pool of organic stores   Input: ranked organic list + ad candidates
Output: ordered organic list    Output: interleaved organic + ad list
```

They run sequentially in the pipeline — ranking happens first, then ads are blended in.
The organic ranker has no knowledge of ads. The ads blender has no knowledge of organic scores.

---

## Where It Lives in the Pipeline

```
restaurantStoreRankingJob           ← organic horizontal ranking (ranked organic list)
         │
         ▼
restaurantStoreDecorationForUserExperienceJob   ← S1/S2/S3 decoration
         │
         ▼
restaurantStoreLayoutProcessingJob  ← constructs HomePageStoreLayoutOutputElements
         │
         ├──────────────────────────────────────────────────────────┐
         │                                                          │
         ▼                                                          ▼
  postProcessingJob                              restaurantStoreAdsBlendingJob
  (vertical ranking)                             (HomepageAdsBlender.blend())
         │                                                          │
         └─────────────────────────────┬────────────────────────────┘
                                       │
                                       ▼
                              outputSerializerJob
```

`HomepageAdsBlender` runs in parallel with `postProcessingJob` (both depend on layout processing).
The final output merges the vertically-ranked carousel list with the ads-blended store lists.

---

## What Gets Blended

`HomepageAdsBlender.blend()` has four separate blending targets, all run in parallel:

```
storeCarouselAdsBlenderJob      ← ads blended INTO each individual store carousel
                                   Surface: SL_CAROUSEL
                                   Method: twoDimensionalSLBlend() (2D: which carousel + which slot)

storeListAdsBlenderJob          ← ads blended into the infinite scroll home feed
  (depends on storeCarouselAdsBlenderJob — dedup against headline carousel ads first)
                                   Surface: SL_HOME_FEED
                                   Method: storeListAdsBlender.blend()

padAdsBlenderJob                ← PAD (Personalized Ad Display) ads blended into collections
                                   Surface: PAD_COLLECTION

immersiveContainerAdsBlenderJob ← ads blended into immersive collection containers
  (depends on padAdsBlenderJob)    Surface: IMMERSIVE_CONTAINER

smartSuggestionsAdsBlenderJob   ← ads blended into smart suggestion collection
                                   Surface: SL_SMART_SUGGESTIONS
                                   Method: oneDimensionalSLBlend()
```

---

## The Blending Algorithms (ICP — Inventory Control Plane)

The algorithm used for any given surface is specified in `HorizontalBlenderConfig.blendingAlgorithm`
and resolved by `BlenderFactory`.

### Three horizontal blender implementations:

**1. GenericHorizontalBlenderV2** (`"generic_1D_blending"`) — the standard blender

Interleaves ads into organic using a cluster sequence pattern.
Config defines the exact slot structure:

```
configuredClusters = [ AdOrganicCluster(adClusterSize=1, organicClusterSize=4) ]
repeatCluster      = AdOrganicCluster(adClusterSize=1, organicClusterSize=4)
startIndex         = 0    ← number of organic items to place before any ads

Result: [AD, O, O, O, O, AD, O, O, O, O, ...]
         slot0 1  2  3  4   5  6  7  8  9
```

More complex example (different slot structure for early vs later positions):
```
configuredClusters = [ (3 ads, 6 organic), (2 ads, 6 organic) ]
repeatCluster      = (1 ad, 6 organic)

Result: [A,A,A, O,O,O,O,O,O, A,A, O,O,O,O,O,O, A, O,O,O,O,O,O, A, ...]
```

Constraints respected per slot:
- **Quality floor**: per-slot minimum and universal minimum (`universalQualityFloor`)
- **Dedupe radius**: ad cannot be adjacent to organic with same `dedupeId` (businessId) within radius N
- **Max ads cap**: `maxNumAds` — total ads placed across all slots
- **Min organic**: won't blend any ads if fewer than `minNumOrganic` organic items

**2. DedupeLastHorizontalBlender** (`"dedupe_last_1D_blending"`) — aggressive dedup

Same cluster-based slotting as above, but with stricter deduplication:
- First occurrence of a `dedupeId` (businessId) wins, ALL subsequent occurrences are dropped
- If an ad store appears organically, the ad is dropped (organic takes precedence)
- No radius — global dedup across the entire output
- Supports pagination cursors (`blendWithPagination`) for infinite scroll surfaces

**3. SPLegacyHorizontalBlender** (`"legacy_sp_1D_blending"`) — legacy sponsored product blender
Legacy path, being migrated toward GenericHorizontalBlenderV2.

---

## Ghost Ads (Lift Experiment Infrastructure)

Every ad in the blended output may carry `ghostAdCandidates`. A ghost ad is an ad that:
- Meets all quality floors
- Is in a lift experiment (campaign has treatment + control split)
- Would have been blended at this slot in the treatment group
- But this user is in the control group, so it is NOT displayed

Ghost ads are attached to the nearest displayed candidate (organic or real ad) and tracked
in Iguazu for iROAS (incremental Return on Ad Spend) measurement. They make it possible to
measure "what revenue was lost by NOT showing this ad" in the control group.

This is a completely separate experiment dimension from organic ranking experiments.

---

## 2D Blending (Carousel-Level + Slot-Level)

`twoDimensionalSLBlend()` is called for store carousels and does blending in two dimensions:

```
DIMENSION 1 (Vertical / carousel-level):
  Which ad carousels get blended into which organic carousels?
  The AdsBlendingDataPlane resolves this based on carousel targeting.
  Each ad store carousel targets specific organic carousel IDs.

DIMENSION 2 (Horizontal / slot-level):
  Within a matched carousel pair, which ad goes in which slot?
  Uses HorizontalBlenderConfig to determine slot positions.
```

The `VerticalBlender` handles dimension 1 (which carousel to blend into).
The `HorizontalBlender` handles dimension 2 (which slot within that carousel).

---

## Key Config Fields

```json
{
  "blending_algorithm": "generic_1D_blending",
  "start_index": 2,                        ← first 2 slots always organic
  "num_ads_limit": 3,                       ← at most 3 ads total
  "min_organic_limit": 5,                   ← don't blend if fewer than 5 organics
  "universal_quality_floor": 0.002,         ← minimum ad quality score
  "dedupe_radius": 1,                       ← no ad adjacent to organic with same businessId
  "configured_slot_sizes": [
    { "ad_cluster_size": 1, "organic_cluster_size": 4 }   ← first cluster
  ],
  "repeated_slot_size": { "ad_cluster_size": 1, "organic_cluster_size": 4 },
  "position_quality_floors": [
    { "start_index": 0, "end_index": 3, "quality_floor": 0.01 }  ← stricter floor in first 3 slots
  ]
}
```

Config is per-surface, stored in runtime JSON, resolved by `BlenderFactory.resolve()`.
This is separate from the organic ranking experiment config (no overlap with DV system for ranking).

---

## The Fundamental Problem for UBP

Organic stores and ad stores are ranked by completely incomparable scoring systems:

```
Organic store score:     Sibyl pCTR/pCVR prediction → finalScore (arbitrary scale, e.g. 0.0-1.0)
Ad store (bid) score:    eCPM / ad quality score (separate Ads ML model, separate scale)

No calibration between these two scores exists today.
```

Because scores are incomparable, organic and ads cannot compete for the same slots. Instead,
the ads system uses a **fixed slotting pattern** (e.g., "slot 0 is always an ad if one is
available") that is independent of the organic ML scores entirely.

Result: a store with `organicScore = 0.95` gets pushed to slot 1 because slot 0 was
reserved for ads, even if the ads system's best ad has `adScore = 0.002` (just above floor).
The user gets a worse organic result so an ad they're unlikely to click can appear first.

**UBP Phase 3 (value function and calibration)** is the solution:
```
calibrate(organicScore) → pAct on a common scale
calibrate(adScore)      → pAct on a common scale

sortedByDescending {
  it.pAct × it.vAct
  where vAct = organic_value_weight (GOV) for organic
              + ad_revenue for sponsored
}
```

When organic and ads compete on the same `pAct × vAct` scale, slot patterns become
obsolete — the best store wins regardless of whether it's organic or sponsored.

---

## What This Means for the Full Pipeline Picture

The complete pipeline, including both organic ranking AND ads blending, looks like this:

```
RETRIEVAL
    │
GROUPING (stores into carousels)
    │
ORGANIC HORIZONTAL RANKING  (Sibyl + business rules, per carousel)
    │
DECORATION + S1/S2/S3 RERANKING  (unobservable post-ranking step)
    │
LAYOUT PROCESSING
    ├────────────────────────────────────────┐
    │                                        │
VERTICAL RANKING (post-processing)      ADS BLENDING (HomepageAdsBlender)
  Organic carousel order                  Insert ad stores into carousels
  Pinned + UR scoring                     and home feed at fixed slot positions
    │                                        │
    └─────────────────────────┬──────────────┘
                              │
                       SERIALIZATION
                       (final response)
```

Two completely separate ranking systems merge at serialization time.
They share no scores, no models, no config.
UBP's end state is to merge them into one unified pipeline with a shared value function.
