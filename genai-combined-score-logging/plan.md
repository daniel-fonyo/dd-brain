# Plan: Log GenAI Combined Score via RankingSignalCollector

**Date**: 2026-03-27 (revised)
**Design doc**: [design.md](design.md) — full architecture, decisions, and rationale

## Context

The `combinedScore` (`finalScore^alpha * similarity^beta`) computed in `GeneratedRecommendationCarouselService.rerankStoreByBlockReranking()` determines store ordering in GenAI carousels but is **discarded after sorting**. Only the original `finalScore` is logged as `HORIZONTAL_ELEMENT_SCORE`. The actual reranking signal is invisible in Snowflake.

## Approach: RankingSignalCollector

Instead of adding per-field wiring (typed field → manual copy → adapter mapping → LoggedValue enum), we build a **universal signal logging pipeline** that works for all homepage carousel types:

1. **`RankingSignalCollector`** — mutable, request-scoped, lives on `ExploreContext`. Any pipeline step calls `collector.record(carouselId, entityId, key, value)`.
2. **`RankingSignalWriter`** — static utility. Each adapter calls it once to flush signals into its logging map.
3. **Proto `map<string, string> ranking_signals`** — new field on `CrossVerticalHomePageFeedEvent`. Lands in Snowflake as a VARIANT column with direct key access (no FLATTEN).
4. **`ContainerEventsGenerator` generic forwarding** — unhandled logging map keys automatically flow into the proto map.

Adding a new ranking signal after this is **one `record()` call** at the source. No downstream changes.

## Two-Phase Implementation

### [Plan 1: Infrastructure](plan-1-infrastructure.md)
- Create `RankingSignalCollector` and `RankingSignalWriter`
- Add collector to `ExploreContext`
- Add proto `ranking_signals` map field
- Wire `StoreCarouselDataAdapter` and `ContainerEventsGenerator`
- **Files**: 2 new + 4 modified (feed-service) + 1 modified (services-protobuf)

### [Plan 2: GenAI Combined Score](plan-2-combined-score.md)
- Add `collector.record()` calls in `GeneratedRecommendationCarouselService`
- **Files**: 1 modified
- Automatically flows through Plan 1 infrastructure to Snowflake

## Key Design Decisions (see [design.md](design.md) for full rationale)

| Decision | Choice | Why |
|---|---|---|
| Where collector lives | `ExploreContext` (not collection types) | Only universal convergence point — works for store, item, deal, announcement carousels |
| How adapters consume | `RankingSignalWriter` utility (not shared interface) | Adapters have no shared base class; writer is a one-line call |
| Entity field needed? | No — signals bypass `StoreEntity` entirely | Writer reads collector → writes directly to logging map |
| Proto type | `map<string, string>` (not `map<string, double>`) | Supports both numbers and strings (model names, strategies) |
| Naming | `rankingSignals` | Avoids collisions with `scoreModifiers`, `rankingMetadata`, `RankingContext` |
| Typed intermediary fields? | No | One `record()` call at source. No `StoreEntity` fields, no `GeneratedRecommendationStoreInfo` fields |

## Execution Order

1. Plan 1 (infrastructure) — can be reviewed/merged independently
2. Plan 2 (genai combined score) — depends on Plan 1
3. Future: wire other adapters (deal, item, announcement) — one line each, as needed

## Future Signals (zero infrastructure changes needed)

After Plan 1, adding any new ranking signal for any carousel type:
```kotlin
// At the source — one line, done
context.rankingSignalCollector.record(carousel.id, entity.id, "new_signal_name", value)
```

Examples:
- Taste affinity score for taste carousels
- Deal relevance score for deal carousels
- SR multiplier explanation string
- Any new ML model output
