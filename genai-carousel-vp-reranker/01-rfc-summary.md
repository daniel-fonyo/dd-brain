# RFC: VP Value Function in GenAI Carousel Horizontal Reranker

**Authors**: Dipali, Ranjan, Yu Zhang
**Date**: 2026-03-20

## Current Reranker Behavior

`reRankStoresByScoreAndSimilarity` in `GeneratedRecommendationCarouselService` uses block reranking:

1. **Read coefficients** from DV per experiment variant: `alpha`, `beta`, `k`
2. **Global sort** by EBR cosine similarity (descending)
3. **Block rerank**: split into chunks of size `k`, within each chunk compute:
   ```
   combinedScore = finalScore^alpha * embeddingScore^beta
   ```
   Sort each chunk by combinedScore descending, stitch chunks back in order.
4. **Return** reordered `LiteStoreCollection`

**Production defaults**: `alpha=0.0`, `beta=1.0`, `k=10` (validated via A/B).
With `alpha=0.0`, `finalScore^0 = 1` — the ranker score drops out entirely. Ranking is 100% EBR similarity today.

### Why blocks, not global sort?

Blocks preserve coarse EBR ordering — a low-similarity store can't leapfrog into a higher block. Only stores already deemed relevant compete for top positions within a block.

## What is EBR?

**Embedding-Based Retrieval**: carousel title + metadata → text embedding; merchant/item profiles → embeddings; k-NN cosine similarity retrieves most relevant stores/items per personalized carousel.

## Proposal

Replace `finalScore` in the block reranker formula with an explicit VP score from raw components already in `ScoreWithFactors`:

```kotlin
// Current
val combinedScore = finalScore.pow(alpha) * similarity.pow(beta)

// Proposed
val pConv       = scoreWithFactors?.rawScorerScores ?: 0.0
val pDasherCost = scoreWithFactors?.expectedDasherCost ?: 1.0
val pCommission = scoreWithFactors?.expectedCommission ?: 1.0
val vpScore     = pConv * pDasherCost * pCommission
val combinedScore = vpScore.pow(alpha) * similarity.pow(beta)
```

### Formula Options Under Discussion

| Option | Formula | Notes |
|--------|---------|-------|
| A | `pConv^alpha * embeddingScore^beta * profitPerConversion` | Keeps embedding score, adds profit multiplier |
| B | Bucket by pConv, rank by finalScore within buckets | No embedding score needed |

Also considering: calibrate SR score first; use rank (not score) for chunk-wise ranking.

## Online A/B Experiment

| Arm | Signal | Description |
|-----|--------|-------------|
| Control | pConv only (`rawScorerScores`) | Clean baseline, no VP multipliers |
| T1 | `pConv * pDasherCost * pCommission` (v1 coefficients) | Full VP formula, first coefficient set |
| T2 | `pConv * pDasherCost * pCommission` (v2 coefficients) | Full VP formula, second coefficient set |

All arms share `beta=1.0` (EBR similarity) and `k=10`. `alpha` set via DV per arm.

## Logging Gap (Our Task)

Currently **not logged**:
- EBR embedding similarity score per store
- The final combined score used for horizontal reranking

These must be added to `cx_cross_vertical_homepage_feed` for evals, training, and debugging.
