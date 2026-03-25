# UBP Spec Directory

This directory contains the formal specification for the Unified Blending Platform.

## Documents

| File | Purpose |
|---|---|
| `rfc.md` | POC RFC: problem statement, goals, non-goals, contracts (0-5), phases, control.json baselines, experiment examples |
| `rfc-abstraction-layer.md` | Abstraction layer RFC for reviewers: full design of `Rankable`, `RankingStep<S>`, `RankingHandler`, `RankingPipeline<S>` |

## Archived

| Directory | Purpose |
|---|---|
| `archive/impl/` | Pre-implementation spec files (9 docs) describing a FeedRow adapter-wrapper design that was superseded by the simpler Rankable interface approach. Kept for historical reference only — **do not use for new work**. |

## Naming Convention (RFC is source of truth)

The RFC (`rfc-abstraction-layer.md`) defines canonical names. The POC branch (`parallel-shadow-ranking`) used stale names that will be updated.

| Concept | RFC Name (canonical) | POC Branch Name (stale) |
|---|---|---|
| Unified interface | `Rankable` | `Scorable` |
| ID method | `rankableId()` | `scorableId()` |
| Ranking step | `RankingStep<S : Enum<S>>` | same |
| Handler (fun interface) | `RankingHandler` | same |
| Step handler | `StepHandler<S>(step, next)` | `StepHandler<S>(step)` + mutable `next` |
| Engine | `RankingPipeline<S : Enum<S>>` | `Ranker<S>` |
| Inter-carousel step type | `CarouselRankStepType { RANK_ALL }` | `VerticalStepType { RANK_ALL }` |
| Inter-carousel RANK_ALL step | `CarouselRankAllStep` | `VerticalRankAllStep` |
| Conversion functions | `toRankableList()` / `toRankableContent()` | `toScorableList()` / `toRankableContent()` |

## Scope

**This RFC covers inter-carousel ranking only.** Intra-carousel ranking (`IntraCarouselRankStepType`, `DiscoveryStore` as `Rankable`) is future work pending Phase 1 results.

## Current Status

- **Phase 1 inter-carousel: POC validated** — shadow mode on sandbox, 99% match
- **Next: rename POC to RFC names, then production shadow validation**
- See `plan.md` in parent directory for full progress tracker
