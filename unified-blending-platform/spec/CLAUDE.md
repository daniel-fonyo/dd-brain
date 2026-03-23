# UBP Spec Directory

This directory contains the formal specification for the Unified Blending Platform.

## Documents

| File | Purpose |
|---|---|
| `rfc.md` | POC RFC: problem statement, goals, non-goals, contracts (0-5), phases, control.json baselines, experiment examples |
| `rfc-abstraction-layer.md` | Abstraction layer RFC for reviewers: full design of `Scorable`, `RankingStep<S>`, `RankingHandler`, `Ranker<S>` — matches shipped Phase 1 implementation |

## Archived

| Directory | Purpose |
|---|---|
| `archive/impl/` | Pre-implementation spec files (9 docs) describing a FeedRow adapter-wrapper design that was superseded by the simpler Scorable interface approach. Kept for historical reference only — **do not use for new work**. |

## Naming Convention (matches shipped code)

| Concept | Name | feed-service location |
|---|---|---|
| Unified interface | `Scorable` | `libraries/platform/.../models/Scorable.kt` |
| Ranking step | `RankingStep<S : Enum<S>>` | `libraries/common/.../ubp/RankingEngine.kt` |
| Handler (chain of responsibility) | `RankingHandler` / `BaseHandler` / `StepHandler` | `libraries/common/.../ubp/RankingEngine.kt` |
| Engine | `Ranker<S : Enum<S>>` | `libraries/common/.../ubp/RankingEngine.kt` |
| Vertical step type | `VerticalStepType { RANK_ALL }` | `libraries/common/.../ubp/VerticalStepType.kt` |
| Vertical RANK_ALL step | `VerticalRankAllStep` | `libraries/common/.../ubp/VerticalRankAllStep.kt` |
| Conversion functions | `toScorableList()` / `toRankableContent()` | `libraries/common/.../ubp/RankableContentConversions.kt` |

## Current Status

- **Phase 1 vertical: shipped** — `Scorable` on 9 types, engine wired in shadow mode
- **Next: shadow validation** — enable `ubp_shadow_vertical_ranking`, validate divergence = 0
- See `plan.md` in parent directory for full progress tracker

## Other documents in unified-blending-platform/

| File | Purpose |
|---|---|
| `plan.md` | Overall project plan, pipeline architecture, migration path, progress tracker |
| `context/northstar.md` | Engineering first principles — the vision and end-state |
| `context/current-system-deep-dive.md` | Deep dive into current feed-service code |
| `context/pivot-analysis.md` | March 2026 pivot: YZ + Dipali transcript synthesis |
| `context/rfc-feedback.md` | Stakeholder conversation notes |
| `context/mle-vertical-contract.md` | Canonical MLE experiment config |
| `context/config-driven-experimentation.md` | How experiment config flows through the pipeline |
| `context/experiment-traffic-industry-research.md` | Industry research for future traffic splitting |
| `context/archive/` | Superseded docs and raw imports (do not use for new work) |
