# GenAI Carousel Reranker — Embedding Score Logging

**Status**: In Progress
**Date**: 2026-03-23
**RFC Authors**: Dipali, Ranjan, Yu Zhang
**Urgency**: High — GenAI V1 live on 60% of orders; need 1+ week of logged data before any experiment can start.

## The Ask

The GenAI carousel horizontal reranker uses an EBR embedding similarity score as the primary ranking signal, but **none of the reranker inputs are logged** to `cx_cross_vertical_homepage_feed`. This blocks offline evals, model training, and debugging. We were asked to add this logging, leveraging past experience wiring VP model predictions into that table.

## What to Log

Agreed with Dipali (meeting 2026-03-20):

| Signal | Granularity | Destination | Priority |
|--------|-------------|-------------|----------|
| EBR embedding similarity score | Per store | `score_modifiers` | **Must have** |
| Alpha (ranker score exponent) | Per carousel | `carousel_details` JSON | **Must have** |
| Beta (embedding score exponent) | Per carousel | `carousel_details` JSON | **Must have** |
| Any future formula param (gamma, etc.) | Per carousel | `carousel_details` JSON | **Auto-logged by design** |

## Design Decisions

### Two channels, matched to data grain

- **`carousel_details`** (field 81, JSON string) — carousel-level data. Set once in container logging, same value for all stores in the carousel. `trackingPayload` on `LiteStoreCollection` flows here automatically. Alpha/beta/future params go here. Adding a new param = adding one line to the `trackingPayload` map in the reranker — zero logging plumbing.
- **`score_modifiers`** (field 59, repeated `ScoreModifier`) — store-level data. Set per child store row. Embedding score goes here via the `LoggedValue` enum pattern (same as `expected_commission`).

### Confirmed from Snowflake data

- `horizontal_element_score` IS populated for GenAI carousels (values like `0.1001`, `0.003`, sometimes `0` when store skips SR).
- `carousel_details` is `{}` for some GenAI traffic — known bug: the Content Systems / EBR fetch path (Path B) drops `trackingPayload` during proto serialization. Path A (direct fetcher) works fine. We need to fix the Path B gap.

### Why not a dedicated proto field for embedding score?

Simpler to use `score_modifiers` — no proto change, no Snowflake schema change, no data eng request. Queryable via `LATERAL FLATTEN` (same pattern Dipali's team already uses for `expected_commission`).

## Why Urgent

- GenAI carousels have their own ranking swimlane that nobody can inspect or improve.
- Embedding score only printed at 5% sample rate in debug logs — not in Iguazu/Data Lake.
- Dipali cannot run offline simulations without this data.
- Every day without logging = another day before experiments can start.

## Meeting Decisions (2026-03-20, Dipali + Daniel)

- **Target table**: `cx_cross_vertical_homepage_feed` (Snowflake via Iguazu). Non-negotiable.
- **Granularity**: One row = one store × one carousel × one consumer. Table already at this grain.
- **Store ranker score**: Already logged via `horizontal_element_score`. For VP experiment users, this is `pConv * profitPerConversion` (not raw pConv). VP formula decomposition is a separate concern.
- Dipali: "Whatever is used to rank should always be logged, even if the formula changes."

## Related Docs

- [01-rfc-and-architecture.md](01-rfc-and-architecture.md) — RFC + codebase reference
- [02-implementation-plan.md](02-implementation-plan.md) — Step-by-step plan
