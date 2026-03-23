# Unified Blending Platform

## Goal
Design a unified blending platform for the feed-service homepage pipeline.

## Key Repos
- feed-service: `/Users/daniel.fonyo/Projects/feed-service`
- services-protobuf: `/Users/daniel.fonyo/Projects/services-protobuf`

## Naming Conventions (Phase 1)

| UBP type | What it is | NOT to be confused with |
|---|---|---|
| `Rankable` | UBP shared interface for ranked content | sdk-core `Scorable` (ML feature extraction) |
| `RankingPipeline` | UBP chain-of-responsibility engine | sdk-p13n `Ranker` (abstract ranking class) |
| `toRankableList()` / `toRankableContent()` | Bridge functions between `RankableContent` and `List<Rankable>` | — |

## Context Files
- `plan.md` — current plan and progress
- `context/` — research and analysis documents
