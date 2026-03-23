# Unified Blending Platform

## Goal
Design a unified blending platform for the feed-service homepage pipeline.

## Key Repos
- feed-service: `/Users/daniel.fonyo/Projects/feed-service`
- services-protobuf: `/Users/daniel.fonyo/Projects/services-protobuf`

## Source of Truth (read these first)
- `context/Unified Blending Platform Vision.md` — Yu Zhang's RFC, the canonical vision
- `context/Unified Blending Platform 1 Pager.md` — executive summary

## Naming Conventions (Phase 1)

| UBP type | What it is | NOT to be confused with |
|---|---|---|
| `Rankable` | UBP shared interface for ranked content | sdk-core `Scorable` (ML feature extraction) |
| `RankingPipeline` | UBP chain-of-responsibility engine | sdk-p13n `Ranker` (abstract ranking class) |
| `RankingStep<S>` | Domain logic contract (items in → items out) | — |
| `RankingHandler` | Infrastructure wrapper (fun interface) | — |
| `toRankableList()` / `toRankableContent()` | Bridge functions between `RankableContent` and `List<Rankable>` | — |

## Context Files
- `plan.md` — current plan and progress tracker
- `context/northstar.md` — engineering first principles
- `context/mle-vertical-contract.md` — canonical MLE experiment config
- `context/config-driven-experimentation.md` — how experiment config flows through the pipeline (code mechanics)
- `context/current-system-deep-dive.md` — how feed-service ranking works today
- `context/horizontal-blending-analysis.md` — ICP ads blending analysis
- `context/horizontal-ranking-analysis.md` — within-carousel ranking analysis
- `context/post-processing-analysis.md` — vertical ranking + fixups analysis
- `context/vertical-ranking-audit-trail.md` — real production debug data
- `context/pivot-analysis.md` — March 2026 strategic pivot
- `context/rfc-feedback.md` — stakeholder conversation notes
- `context/experiment-traffic-industry-research.md` — industry research for future traffic splitting
- `context/archive/` — superseded docs and raw imports (do not use for new work)
