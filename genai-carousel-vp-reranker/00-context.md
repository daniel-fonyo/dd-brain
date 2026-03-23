# GenAI Carousel Reranker — Embedding Score Logging

**Status**: In Progress
**Date**: 2026-03-23
**RFC Authors**: Dipali, Ranjan, Yu Zhang
**Urgency**: High — GenAI V1 live on 60% of orders; need 1+ week of logged data before any experiment can start.

## The Ask

The GenAI carousel horizontal reranker uses an EBR embedding similarity score as the primary ranking signal, but **none of the reranker inputs are logged** to `cx_cross_vertical_homepage_feed`. This blocks offline evals, model training, and debugging. We were asked to add this logging, leveraging past experience wiring VP model predictions into that table.

## What to Log

Agreed with Dipali (meeting 2026-03-20):

| Signal | Granularity | Priority |
|--------|-------------|----------|
| EBR embedding similarity score | Per store × carousel × consumer | **Must have** |
| Alpha (ranker score exponent, from DV) | Per carousel | **Must have** |
| Beta (embedding score exponent, from DV) | Per carousel | **Must have** |
| Any future formula param (gamma, etc.) | Per carousel | **Design for auto-logging** |

**Key design requirement**: The formula will evolve — Dipali plans to add VP multipliers, possibly a gamma coefficient, possibly entirely new formulas. Whatever params are used to rank must automatically be logged without requiring new backend plumbing each time.

The store ranker score (`finalScore`) is already logged as `horizontal_element_score` — need to confirm it populates for GenAI programmatic carousels specifically.

## Why Urgent

- GenAI carousels have their own ranking swimlane that nobody else can inspect or improve.
- The embedding score is only printed at 5% sample rate in debug logs — not produced to Iguazu/Data Lake.
- Dipali cannot run offline simulations to test formula changes without this data.
- Every day without logging = another day before experiments can start.

## Meeting Decisions (2026-03-20, Dipali + Daniel)

- **Target table**: `cx_cross_vertical_homepage_feed` (Snowflake via Iguazu). Non-negotiable.
- **Granularity**: One row = one store × one carousel × one consumer. Table already at this grain.
- **How to log**: Left to Daniel's judgment. Dipali suggested JSON dump or extensible map for formula params so future changes are self-logging.
- **Store ranker score**: Already logged via `horizontal_element_score`. For VP experiment users, this is `pConv * profitPerConversion` (not raw pConv). VP formula decomposition is a separate concern.
- Dipali: "Whatever is used to rank should always be logged, even if the formula changes."

## Related Docs

- [01-rfc-and-architecture.md](01-rfc-and-architecture.md) — RFC + codebase reference
- [02-implementation-plan.md](02-implementation-plan.md) — Step-by-step plan
