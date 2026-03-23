# GenAI Carousel VP Reranker — Overview

**Status**: In Progress
**Date**: 2026-03-23
**Authors (RFC)**: Dipali, Ranjan, Yu Zhang
**Ask**: Log embedding scores used in GenAI horizontal carousel ranking; support VP value function experiment.

## Problem

The GenAI carousel horizontal reranker uses EBR embedding similarity scores to rank stores within each carousel. These scores are **not logged** to `cx_cross_vertical_homepage_feed`, which blocks:

1. Offline evals of ranking changes
2. Model training on ranking signals
3. Debugging production ranking behavior

Additionally, the reranker's VP (Variable Profit) score slot is currently dormant (`alpha=0.0`), and an upcoming experiment needs clean logging of all component signals.

## Our Role

We were asked to help add embedding score logging to `cx_cross_vertical_homepage_feed`, leveraging past experience adding VP model predictions to that table.

## Key Deliverables

1. **Proto change**: Add `embedding_score` field to `CrossVerticalHomePageFeedEvent` in `services-protobuf`
2. **Logging plumbing**: Wire embedding score through `ContainerEventsGenerator` / `LoggedValue` enum in feed-service
3. **Verify**: Scores appear in Snowflake `cx_cross_vertical_homepage_feed` table

## Related Docs

- [01-rfc-summary.md](01-rfc-summary.md) — RFC distillation
- [02-architecture.md](02-architecture.md) — Codebase map of ranking + logging paths
- [03-implementation-plan.md](03-implementation-plan.md) — Step-by-step plan
