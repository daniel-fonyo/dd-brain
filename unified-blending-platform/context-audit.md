# UBP Context Directory — Audit Report

**Date**: 2026-03-22
**Source of truth**: `Unified Blending Platform Vision.md`, `Unified Blending Platform 1 Pager.md`, `poc-generic-ranking.md`

---

## Executive Summary

28 files in `context/` totaling ~1.6MB. Of those, ~1.4MB is raw imports from Google Docs that are already distilled into smaller analysis docs. Multiple naming conventions for the same abstractions (`Scorable` vs `VerticalComponent` vs `FeedRow`). The March 2026 pivot (traffic management first, simplified MLE contract) is documented in `pivot-analysis.md` but NOT propagated to `plan.md`, `northstar.md`, or the MLE contract docs. Any Claude session that reads these docs will get contradictory guidance on naming, step decomposition, config schema, and priorities.

---

## 1. Naming Conflicts (3 competing schemes)

The core abstraction — a unified interface wrapping all carousel types — has 3 names:

| Source | Interface | Step | Engine |
|---|---|---|---|
| `poc-generic-ranking.md` (SOT) | `Scorable` | `RankingStep<S>` | `Ranker<S>` |
| `plan.md`, `northstar.md`, `design-patterns-and-contract.md` | `VerticalComponent` | `VerticalProcessor` | `VerticalRankingEngine` |
| `one-pager-abstraction-proposal.md`, `one-pager-diagrams.md` | `FeedRow` | `FeedRowRankingStep` | `FeedRowRanker` |

**Impact**: Claude asked to implement the abstraction will produce code using whichever naming it reads last — or hallucinate a hybrid.

**Resolution**: Pick one naming convention. `poc-generic-ranking.md` is SOT — use `Scorable` / `RankingStep` / `Ranker` as the canonical names. Update `plan.md` and `northstar.md` to match. Delete or archive docs that use `FeedRow` naming.

---

## 2. Step Decomposition Conflict (4-step vs 2-step)

| Source | Steps |
|---|---|
| `plan.md` | 4: `MODEL_SCORING`, `MULTIPLIER_BOOST`, `DIVERSITY_RERANK`, `FIXED_PINNING` |
| `design-patterns-and-contract.md`, `northstar.md`, `mle-vertical-contract.md`, `experiment-config-contract.md`, `mle-experiment-guide.md` | 2: `MODEL_SCORING`, `BOOST_AND_RANK` |
| `poc-generic-ranking.md` (SOT) | 1: `RANK_ALL` (Phase 1), granular steps as future extension |

**Impact**: `plan.md` — the first doc Claude reads for implementation tasks — uses a 4-step model that no other doc agrees with. The Phase 1 implementation steps (Step 1-6 in `plan.md`) are designed around the 4-step model.

**Resolution**: Align `plan.md` to the 2-step model from the contract docs. The 4-step model was an early iteration that was abandoned.

---

## 3. Config Schema Conflicts (3+ JSON shapes)

The MLE experiment JSON has at least 3 different shapes across docs:

**Shape A — Flat (design-patterns-and-contract.md, mle-contract-interview.md):**
```json
{ "p_act": { "predictor": "...", "model": "..." }, "v_act": { "gov": 1.0 }, "boost_weights": {} }
```

**Shape B — Nested pipeline (mle-vertical-contract.md, experiment-config-contract.md):**
```json
{ "value_function": { "p_act": { "predictor_name": "..." } }, "vertical_pipeline": { "steps": [...] } }
```

**Shape C — step_params override (experiment-config-contract.md, mle-experiment-guide.md):**
```json
{ "extends": "control", "step_params": { "score": { "primary_model": "..." } } }
```

**Impact**: An MLE writing an experiment config gets contradictory guidance on what the JSON looks like.

**Resolution**: Canonicalize on one schema. Delete docs that show superseded schemas.

---

## 4. Pivot Not Propagated

`pivot-analysis.md` documents a March 2026 directional shift:
- **Old**: Complex config-driven engine where MLEs author pipeline step sequences
- **New**: Simplified experiment declaration (model + layer + traffic %) with per-layer traffic management as #1 priority

These docs still reflect the OLD direction:
- `plan.md` — priorities are engine-first, no traffic management
- `northstar.md` — shows full MLE pipeline configs as the "done" state
- `mle-vertical-contract.md` — full pipeline declaration
- `experiment-config-contract.md` — full pipeline declaration
- `mle-experiment-guide.md` — 7 cases all using old schema

**Impact**: Claude will plan work around the pre-pivot priorities (engine first) instead of post-pivot (traffic management first).

---

## 5. Raw Imports — 1.4MB of Bloat

These are unprocessed pastes from Google Docs/Notion. Every one has already been distilled into a smaller analysis doc:

| Raw Import | Size | Lines | Already Distilled In |
|---|---|---|---|
| `Homepage Ads Blending.md` | 672KB | 115 | `horizontal-blending-analysis.md` |
| `Homepage Blender V1.md` | 565KB | 218 | `horizontal-blending-analysis.md`, `post-processing-analysis.md` |
| `Carousel Ranking Model.md` | 182KB | 363 | `current-system-deep-dive.md`, `horizontal-ranking-analysis.md` |
| `[RFC] Universal Ranker Modeling.txt` | 52KB | 804 | **Duplicate** of `RFC Universal Ranking Model.md` |
| `RFC Universal Ranking Model.md` | 51KB | 804 | `current-system-deep-dive.md` |
| `Homepage Ranking.md` | 21KB | 275 | `horizontal-ranking-analysis.md` |
| `Store Ranker Feed Service Details.md` | 23KB | 843 | `horizontal-ranking-analysis.md` |

**Total**: ~1.57MB of raw imports that duplicate distilled content.

**Impact**: If Claude tries to read these for context, it wastes tokens on unstructured content that contradicts the more precise analysis docs.

**Resolution**: Move to `context/archive/raw-imports/` or delete entirely.

---

## 6. Superseded / Redundant Docs

### Should DELETE (content fully superseded):

| Doc | Why |
|---|---|
| `problem-and-approach.md` | Everything in it is covered better in `northstar.md` + Vision + 1 Pager. Same problem statement, same 4-layer pipeline, same "why start with vertical" — shorter but adds nothing unique. |
| `[RFC] Universal Ranker Modeling.txt` | Byte-for-byte duplicate of `RFC Universal Ranking Model.md` |

### Should ARCHIVE (content superseded but has historical value):

| Doc | Why |
|---|---|
| `one-pager-abstraction-proposal.md` | Uses old `FeedRow` naming. Pre-pivot proposal. The design patterns appendix is 300+ lines of textbook GoF/Feathers references that are educational but not operational context. Superseded by `poc-generic-ranking.md` + `design-patterns-and-contract.md`. |
| `one-pager-diagrams.md` | Companion to above. All diagrams use old `FeedRow` naming. |
| `poc-article-patterns.md` | Academic mapping of a Medium article to GoF patterns. Not UBP-specific. |

### Should CONSOLIDATE:

| Docs | Issue | Action |
|---|---|---|
| `mle-vertical-contract.md` + `experiment-config-contract.md` + `mle-experiment-guide.md` + `mle-contract-interview.md` | 4 docs all describing the MLE contract with conflicting schemas | Merge into one canonical `mle-contract.md` |
| `design-patterns-and-contract.md` + `northstar.md` | Significant overlap in architecture description (interfaces, engine, value function, N×M model) | Merge — `northstar.md` is the better doc; `design-patterns-and-contract.md` adds the pattern analysis which could be a section |

---

## 7. Docs With Stale Sections

| Doc | Stale Section | Issue |
|---|---|---|
| `plan.md` | Step types table (lines 79-84) | Lists 4 steps; current model is 2 |
| `plan.md` | Implementation Steps 1-6 | Designed around 4-step model and engine-first priority |
| `plan.md` | Progress Tracker | Still shows 4 vertical steps + horizontal + Phase 3 — doesn't reflect pivot |
| `northstar.md` | Section 7 "Config Contract" | Shows `vertical_pipeline.steps` with `predictor_ref` — conflicts with simplified contract |
| `mle-contract-interview.md` | Bottom section (lines 323-331) | Raw unformatted scratchpad notes ("VP", "Exploration ranker", "How to validate values from contract?") |
| `rfc-feedback.md` | Entire doc | Labeled "temp doc" — feedback is valuable but should be folded into `pivot-analysis.md` or a stakeholder-notes doc |

---

## 8. Good Docs (keep as-is)

| Doc | Why |
|---|---|
| `Unified Blending Platform Vision.md` | SOT — Yu Zhang's RFC |
| `Unified Blending Platform 1 Pager.md` | SOT — executive summary |
| `poc-generic-ranking.md` | SOT — POC engine design |
| `current-system-deep-dive.md` | High quality ground-truth reference for existing system |
| `horizontal-blending-analysis.md` | Clean analysis of ICP ads blending; replaces raw imports |
| `horizontal-ranking-analysis.md` | Clean analysis of within-carousel ranking |
| `post-processing-analysis.md` | Clean analysis of vertical ranking + fixups |
| `vertical-ranking-audit-trail.md` | Real debug data from production — irreplaceable |
| `pivot-analysis.md` | Documents the strategic pivot — critical context |
| `experiment-traffic-industry-research.md` | Industry research for traffic management design |

---

## 9. Recommended File Structure After Cleanup

```
unified-blending-platform/
├── CLAUDE.md
├── plan.md                          ← UPDATE: align with pivot + 2-step model
└── context/
    ├── source-of-truth/
    │   ├── Unified Blending Platform Vision.md
    │   ├── Unified Blending Platform 1 Pager.md
    │   └── poc-generic-ranking.md
    ├── system-analysis/
    │   ├── current-system-deep-dive.md
    │   ├── horizontal-blending-analysis.md
    │   ├── horizontal-ranking-analysis.md
    │   ├── post-processing-analysis.md
    │   └── vertical-ranking-audit-trail.md
    ├── design/
    │   ├── northstar.md              ← UPDATE: merge design-patterns, fix config schema
    │   ├── mle-contract.md           ← NEW: consolidation of 4 MLE contract docs
    │   └── experiment-traffic-industry-research.md
    ├── stakeholder/
    │   ├── pivot-analysis.md
    │   └── rfc-feedback.md           ← CLEAN UP: remove "temp" label, format notes
    └── archive/
        ├── raw-imports/              ← 7 raw Google Doc pastes (1.4MB)
        ├── one-pager-abstraction-proposal.md
        ├── one-pager-diagrams.md
        └── poc-article-patterns.md
```

---

## 10. Actions Taken (2026-03-22)

1. **Deleted** `problem-and-approach.md` and `[RFC] Universal Ranker Modeling.txt` (duplicates)
2. **Archived** 6 raw imports to `archive/raw-imports/` + 3 superseded docs to `archive/`
3. **Rewrote `plan.md`** — aligned with poc-generic-ranking naming (Scorable/RankingStep/Ranker), Phase 1 RANK_ALL model, traffic splitting moved to future
4. **Consolidated** `mle-vertical-contract.md` + `experiment-config-contract.md` + `mle-experiment-guide.md` + `mle-contract-interview.md` → one canonical `mle-contract.md` (flat schema, open questions preserved)
5. **Updated `northstar.md`** — merged design-patterns-and-contract.md content, fixed naming + config schema, deleted source doc
6. **Standardized naming** across plan.md, northstar.md, rfc-feedback.md to Scorable/RankingStep/Ranker
7. **Cleaned up** rfc-feedback.md (removed "temp" label, updated component names)
8. **Updated** CLAUDE.md with new file structure and naming convention
