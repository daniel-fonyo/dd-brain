# Vertical Ranking Audit Trail

> Reference doc for understanding what flows through the vertical blending pipeline and what doesn't.
> Companion to the `[dfonyo-debug]` logs added in `feat/dfonyo-vertical-ranking-debug-logs`.

---

## The Full Vertical Ranking Flow

```
reOrderGlobalEntitiesV2()
    │
    ├── rankAndDedupeContent()
    │       └── rankAndMergeContent()
    │               └── EntityRankerConfiguration.rank()
    │                       ├── Sibyl scoring → ScoreBundle (ALL 9 types get a score)
    │                       └── BlendingUtil.blendBundle()
    │                               ├── [blend-filter] StoreCarousel → INCLUDED
    │                               ├── [blend-filter] ItemCarousel  → INCLUDED (if itemCarouselBlendingEnabled)
    │                               ├── [blend-filter] DealCarousel  → SKIPPED (else → null)
    │                               ├── [blend-filter] CollectionV2  → SKIPPED
    │                               ├── [blend-filter] StoreCollection → SKIPPED
    │                               ├── [blend-filter] MapCarousel   → SKIPPED
    │                               ├── [blend-filter] ReelsCarousel → SKIPPED
    │                               ├── [blend-filter] ItemCollection → SKIPPED
    │                               ├── [blend-filter] StoreEntity   → SKIPPED
    │                               │
    │                               ├── [blend-score] per included entity:
    │                               │       calibration × intent × boost_weight → final_score
    │                               │
    │                               └── [blend-diversity] optional greedy Rx/non-Rx rerank
    │
    ├── [final-sort] ScoreBundle.getSortedList() — all 9 types sorted by finalScore desc
    │       StoreCarousel: blended score
    │       ItemCarousel:  blended score (if enabled) else raw Sibyl
    │       DealCarousel:  raw Sibyl score — no calibration, no boost, no diversity
    │       CollectionV2:  raw Sibyl score
    │       ... all others: raw Sibyl score
    │
    ├── [nv-pin] updateSortOrderOfNVCarousel() — post-checkout NV carousel sort order override
    │
    ├── [pad-pin] updateSortOrderOfTasteOfDashPass() — PAD (TasteOfDashPass) moved to position 3
    │
    ├── [member-pricing] updateSortOrderOfMemberPricing() — member pricing carousels → position 0
    │
    └── [pipeline-exit] rankWithImmersivesV2() — color bleed + immersive spacing → final sort orders
```

---

## The Core Problem: Incomparable Scores

`BlendingUtil` only blends `StoreCarousel` and `ItemCarousel`. All other types get raw Sibyl scores.

After blending, the default boost multiplier is **3.75×** (and up to **8.75×** for some NV verticals).

A `DealCarousel` with Sibyl score `0.1` competes against a `StoreCarousel` with Sibyl score `0.02` that
gets boosted to `0.175`. The deal carousel loses despite scoring 5× higher before blending.

The scores are not on a comparable scale. The final sort pretends they are.

---

## What the Debug Logs Capture

### `[dfonyo-debug][blend-filter]`
One log per entity entering `blendBundle`. Shows:
- `carousel_id`, `type`, `included` (true/false), `reason`, `pre_score`
- Answers: what made it into the blender?

### `[dfonyo-debug][blend-score]`
Three logs per blended entity (one per step):
- `step=calibration`: `vertical_id`, `pre_cal_score`, `cal_mult`, `post_cal_score`
- `step=intent`: `intent_mult`, `intent_pred_score`, `post_intent_score`
- `step=boost`: `boost_weight`, `final_score`
- Answers: what multiplier was applied at each step?

### `[dfonyo-debug][blend-diversity]`
One log with full reranked order (only if reranking enabled):
- Each position: `carousel_id`, `is_rx`, `final_score`, `diversity_score`
- Answers: how did Rx/non-Rx diversity reranking change the order?

### `[dfonyo-debug][blend-exit]`
One log at end of `blendBundle` showing all final scores:
- All blended entities in score order
- Answers: what came out of the blender and in what order?

### `[dfonyo-debug][final-sort]`
One log from `ScoreBundle.getSortedList()` showing ALL carousel types sorted:
- Every carousel: `type`, `carousel_id`, `final_score`, `position`
- Answers: what is the full vertical ordering before fixups?

### `[dfonyo-debug][nv-pin]`
Logs from `updateSortOrderOfNVCarousel()`:
- Which carousels had sort order overridden, and to what position
- Answers: what got NV-pinned and where?

### `[dfonyo-debug][pad-pin]`
Log from `updateSortOrderOfTasteOfDashPass()`:
- Whether PAD was moved, from what sort order to 3
- Answers: was PAD repositioned?

### `[dfonyo-debug][member-pricing]`
Log from `updateSortOrderOfMemberPricing()`:
- Which carousels got bumped to position 0
- Answers: did member pricing override anything?

### `[dfonyo-debug][pipeline-entry]`
Log at start of `reOrderGlobalEntitiesV2()`:
- Count and IDs of all carousel types entering the pipeline
- Answers: what was the full input set?

### `[dfonyo-debug][pipeline-exit]`
Log at end of `reOrderGlobalEntitiesV2()` after all fixups:
- Final sort order of all carousels
- Answers: what is the final page order?

---

## How to Use the Logs

```bash
# Follow one request end to end
grep "\[dfonyo-debug\]" logs | grep "consumer=<consumer_id>"

# See what got filtered out of blending
grep "\[blend-filter\]" logs | grep "included=false"

# See the full score progression for one carousel
grep "\[blend-score\]" logs | grep "carousel_id=<id>"

# See what the silent overrides did
grep -E "\[nv-pin\]|\[pad-pin\]|\[member-pricing\]" logs

# Compare pre-fixup vs post-fixup order
grep -E "\[final-sort\]|\[pipeline-exit\]" logs | grep "consumer=<consumer_id>"
```

---

## feed-service branch
`feat/dfonyo-vertical-ranking-debug-logs`

## Files modified
- `VerticalBlending.kt` — blend-filter, blend-score, blend-diversity, blend-exit
- `ScoreBundle.kt` — final-sort
- `NonRankableHomepageOrderingUtil.kt` — nv-pin, pad-pin, member-pricing
- `DefaultHomePagePostProcessor.kt` — pipeline-entry, pipeline-exit
