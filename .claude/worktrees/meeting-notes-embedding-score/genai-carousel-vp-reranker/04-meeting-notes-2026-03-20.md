# Meeting Notes: Dipali + Daniel — Embedding Score Logging

**Date**: 2026-03-20
**Attendees**: Dipali Ranjan, Daniel Fonyo

## Context

GenAI V1 has rolled out to Tier 1 + Tier 2 submarkets — changes affect ~60% of all DoorDash orders. GenAI carousels are a special collection type with their own retrieval (EBR) and their own horizontal ranking system that bypasses the regular store ranker.

## Key Points from Dipali

### The Problem
- The embedding score (EBR cosine similarity between carousel title and store embedding) is **not logged anywhere queryable** — only printed at 5% sample rate in logs, not produced to Iguazu/Data Lake.
- Nobody has visibility into how GenAI carousels are ranked horizontally. The GenAI team effectively has their own swimlane on ranking that nobody else can inspect or improve.
- Without logging, Dipali **cannot run offline simulations** to test new alpha/beta values or new formulas. She needs at least 1 week of data to do any eval.
- **Urgency**: The longer we wait, the more time is wasted before they can launch experiments.

### What the Embedding Score Is
- Similarity between carousel title embedding and store embedding.
- Dipali believes it's computed **online (real-time)**, not offline — based on what Joni told her.
- Conceptually: embedding score = **relevance signal**, store ranker score = **conversion signal (pConv)**. Normally these would be combined in one model, but because GenAI is new, they're separate.

### What Needs to Be Logged
Minimum viable set agreed on:
1. **Embedding score** (per store, per carousel, per consumer)
2. **Alpha** (exponent for ranker score)
3. **Beta** (exponent for embedding score)

Nice-to-haves discussed:
- **Final score** (the store ranker score used in the formula) — already logged via `horizontal_element_score`, but confirm it shows up for GenAI carousels specifically.
- VP components if we're already in there.

### Granularity
- One row = one carousel + one store (per consumer). The `cx_cross_vertical_homepage_feed` table is already at this granularity.

### Where to Log
- Must be in `cx_cross_vertical_homepage_feed` table (Snowflake via Iguazu). Other locations would make downstream work harder.
- Dipali doesn't have a strong preference on *how* (new proto fields vs `score_modifiers` vs JSON). Left to Daniel's judgment.

### Scalability Concern
- Dipali plans to **change the formula** in upcoming experiments (e.g., add VP profit multiplier, possibly add gamma parameter, possibly create entirely new formula).
- She wants a logging approach where new parameters can be added without requiring another round of plumbing work each time.
- Suggestion: consider a JSON dump or extensible map approach so future formula changes (e.g., adding gamma for VP) don't require re-plumbing.
- Dipali: "Whatever is used to rank should always be logged, even if the formula changes."

### Relationship to Store Ranker Score
- `finalScore` in the reranker formula comes from the store ranker (SR).
- For users in the VP experiment, SR score is already overridden with VP score (`pConv * profitPerConversion`), so `finalScore` is **not raw pConv** — it's already VP-multiplied.
- The raw pConv and VP components are a separate concern Dipali will handle in the RFC.

## Agreed Next Steps

1. Daniel pulls down the code, gets a feel for the effort, and responds in the channel with an estimate.
2. Start with alpha, beta, embedding score as the minimum logging set.
3. Consider extensible approach for future formula parameters.

## Implications for Implementation Plan

Updates to `03-implementation-plan.md`:
- **Log alpha and beta** in addition to embedding score — these are the DV-controlled coefficients.
- **Consider extensible format**: A JSON field or map-based approach (like `carousel_details` pattern) would let Dipali add future formula params (gamma, etc.) without new proto fields each time.
- **Confirm `horizontal_element_score` populates** for GenAI programmatic carousels — Dipali wasn't sure if the store ranker score shows up there for GenAI collections.
- **Urgency**: Ship ASAP — they need 1+ week of data before running any experiment.
