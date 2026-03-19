# UBP Spec Directory

This directory contains the formal specification and implementation plan for the
Unified Blending Platform.

## Documents in this directory

| File | Purpose |
|---|---|
| `rfc.md` | Platform contracts (FeedRow/RowItem interfaces, Experiment Config JSON, Value Function, RankingStep, Trace Event) |
| `impl/00-overview.md` | Implementation order, dependencies between parts, guiding principles |
| `impl/01-shadow-infra.md` | Shadow traffic path for safe validation before any real traffic migrates |
| `impl/02-feed-row-interface.md` | FeedRow interface + 9 adapters (Adapter pattern) |
| `impl/03-ranking-step.md` | FeedRowRankingStep interface + 4 step implementations (Strategy pattern) |
| `impl/04-ranker-engine.md` | FeedRowRanker engine + step registry (Facade + Chain of Responsibility) |
| `impl/05-config.md` | UnifiedExperimentConfig data classes + UbpRuntimeUtil (Factory Method + Prototype) |
| `impl/06-wiring.md` | Wire engine into DefaultHomePagePostProcessor and DefaultHomePageStoreRanker |
| `impl/07-tracing.md` | Auto-trace infrastructure + ubp_feed_row_ranking_trace.proto (Observer pattern) |

## Other documents in unified-blending-platform/

| File | Purpose |
|---|---|
| `plan.md` | Overall project plan, pipeline architecture, migration path, progress tracker |
| `context/northstar.md` | Engineering first principles — the vision and end-state |
| `context/problem-and-approach.md` | Root cause analysis and the mental model for the solution |
| `context/Unified Blending Platform Vision.md` | Full vision doc with all pain point themes |
| `context/mle-vertical-contract.md` | MLE-facing contract for vertical blending experiments |
| `context/experiment-config-contract.md` | Experiment config JSON examples |
| `context/design-patterns-and-contract.md` | Design patterns used + value function |
| `context/current-system-deep-dive.md` | Deep dive into current feed-service code |
