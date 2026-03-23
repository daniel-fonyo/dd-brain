# Unified Blending Platform

## Goal
Design a unified blending platform for the feed-service homepage pipeline.

## Key Repos
- feed-service: `/Users/daniel.fonyo/Projects/feed-service`
- services-protobuf: `/Users/daniel.fonyo/Projects/services-protobuf`

## Source of Truth (read these first)
- `context/Unified Blending Platform Vision.md` — Yu Zhang's RFC, the canonical vision
- `context/Unified Blending Platform 1 Pager.md` — executive summary
- `context/poc-generic-ranking.md` — POC engine design (Scorable/RankingStep/Ranker)

## Naming Convention (from poc-generic-ranking.md)
- `Scorable` — unified interface for all rankable types
- `RankingStep<S>` — domain logic contract (items in → items out)
- `RankingHandler` — infrastructure wrapper (metrics, conditions, shadow)
- `Ranker<S>` — config-driven engine that assembles handler chains

## Context Files
- `plan.md` — current plan and progress tracker
- `context/northstar.md` — engineering first principles
- `context/mle-contract.md` — canonical MLE experiment config
- `context/current-system-deep-dive.md` — how feed-service ranking works today
- `context/horizontal-blending-analysis.md` — ICP ads blending analysis
- `context/horizontal-ranking-analysis.md` — within-carousel ranking analysis
- `context/post-processing-analysis.md` — vertical ranking + fixups analysis
- `context/vertical-ranking-audit-trail.md` — real production debug data
- `context/pivot-analysis.md` — March 2026 strategic pivot
- `context/rfc-feedback.md` — stakeholder conversation notes
- `context/experiment-traffic-industry-research.md` — industry research for future traffic splitting
- `context/archive/` — superseded docs and raw imports (do not use for new work)
