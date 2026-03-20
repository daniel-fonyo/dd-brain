# Unified Blending Platform — Problem & Approach

## The Problem

The DoorDash homepage started as a single-vertical product (just Restaurants). Over time it grew to serve multiple content types on the same page: Rx stores, NV (grocery/convenience) stores, item carousels, ads, deals, DashPass merchandising. Each of these was built independently by different teams with their own ranking logic.

The result: **there is no single system deciding what goes where on the page**. Instead there are 4-5 separate systems each doing their own thing, stitched together with heuristics:

- Organic stores ranked by one Sibyl model
- Ads inserted after organic ranking by a separate blender
- NV stores boosted by hardcoded multipliers
- Merchandising carousels pinned by campaign rules
- Post-ranking fixups (NV position, PAD position, member pricing) that silently override everything before them

Nobody can answer: "Is this NV carousel worth more to the user than this Rx carousel at position 3?" — because the scores are on completely different scales and computed by different systems.

---

## The Feed-Service Reality

The homepage pipeline runs in 4 layers:

```
RETRIEVAL → GROUPING → HORIZONTAL RANKING → VERTICAL RANKING
```

UBP Phase 1 focuses on **vertical ranking** (Layer 4) — which carousel row goes at which position on the page.

Today that lives in `DefaultHomePagePostProcessor.reOrderGlobalEntitiesV2()`. It does 7 stages:

1. Split carousels into "pinned" vs "rankable"
2. Call Sibyl to score the rankable ones
3. Apply calibration × intent × boost weight multipliers (`BlendingUtil.blendBundle()`)
4. Optionally diversity-rerank (`rerankEntitiesWithDiversity()`)
5. Enforce pinned positions (`BoostingBundle.boosted()`)
6. Sort by final score
7. Post-ranking fixups (NV pin, PAD to position 3, member pricing, gap rules)

**The core problems:**

- Those 7 stages are hardcoded. No config drives which stages run or in what order.
- Params for each stage live in 10+ places: 3 runtime JSON files, 5+ DV keys, hardcoded constants.
- Stage 7 runs *outside* the pipeline entirely — silently overrides everything before it with no logging or traceability.
- Each carousel type (`StoreCarousel`, `ItemCarousel`, `DealCarousel`, etc.) is a separate class with no shared interface. Operating on "all carousels" means writing the same logic 9 times.
- Changing one experiment param (e.g. NV boost weight) requires touching multiple files, multiple DVs, and senior MLE review. 2-3 weeks for what should take 2-3 days.

---

## The Mental Model for the Solution

Replace the hardcoded pipeline with a **config-driven DAG**.

Instead of:
```kotlin
fun rank() {
    score()     // calls Sibyl
    blend()     // multipliers
    diversity() // rerank
    pin()       // enforce positions
    fixups()    // silently overrides everything
}
```

You have a JSON that MLEs write:
```json
"steps": [
  { "type": "MODEL_SCORING",  "params": { "predictor_ref": "p_act" } },
  { "type": "BOOST_AND_RANK", "params": { "boost_by_position_enabled": false } }
]
```

The engine reads the JSON, loops the steps, dispatches to registered processors. MLE changes the JSON — no code change, no PR. BE engineer adds a new `Processor` class when a new step type is needed — once. After that, anyone can use it via config.

**Three things that must exist:**

1. **`VerticalComponent` interface** — wraps all 9 carousel types behind one interface so processors work on `List<VerticalComponent>`, never branching on type.

2. **`VerticalProcessor` interface** — each stage is a registered class taking `(components, context, params)`. Params come from JSON, not internal DV reads. This is what makes processors reusable across experiments.

3. **`VerticalRankingEngine`** — reads step list from config, dispatches to processor registry, auto-traces after every step. Zero business logic.

The existing code (`BlendingUtil`, `BoostingBundle`, `entityScorer`) doesn't go away — it gets wrapped inside processor classes. The logic is the same; what changes is how it's wired and configured.

---

## Why Start With Vertical

Horizontal ranking (within-carousel store ordering) is tackled second because:
- Vertical has more fragmented heuristics (7 stages vs 3)
- `VerticalBlendingConfig` JSON is already a working config seam to extend from
- Carousel ordering is lower risk to users than within-carousel ordering
- Immediately unblocks NV, PAD, and LCM experiments blocked on HP bandwidth
