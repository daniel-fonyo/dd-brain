# UBP Pivot Analysis — March 2026

> Synthesis of YZ + Dipali conversations with existing UBP context.
> This document captures the pivot from "complex config-driven engine" to
> "simplified experiment declaration + per-layer traffic management."

---

## Source Conversations

- **Yu Zhang (YZ)** — UBP scope, experiment traffic management, collaboration model
- **Dipali Ranjan** — MLE experiment workflow, control ownership, GenAI carousel gaps, code messiness

---

## Key Findings

### From YZ

1. **Vertical blending is the most mature work stream** — confirms starting there.
2. **The real MLE contract = model + weights.** "The key part when this just comes back to they have a different model, and they want to test different set of weights."
3. **Per-layer experiment traffic management is the #1 pain point.**
   - Current system: 4 consumer cohorts → priority list → dummy DVs for traffic splitting → reserve segments → waterfall to next experiment.
   - "Right now, it's nasty work." Changing one experiment in the waterfall breaks all downstream.
   - Desired state: "I'm just telling you, hey, this experiment is taking that much traffic, I'm done."
4. **YZ expects Daniel to redesign the experiment traffic system.** "I'm expecting you, maybe you can just overwrite this entire... redesign this entire experiment, how they should be navigating traffic."
5. **Start from what you can handle, not from MLE wishlist.** "What you feel comfortable you can do. Let's go starting from there."
6. **This is Daniel's own work.** "This experiment management thing is just your own. Everyone is gonna working on the old part, at some point, we just switch to you."
7. **Frank/Harsha predictor splitting** — parallel work, incorporate if useful, ignore if not.

### From Dipali

1. **Experiments = model swaps.** "Predictor is done, you don't have to touch the predictor. Predictor is one time."
2. **Calibration is a workflow, not a contract feature.** Model ready → calibrate → parameter tune (2-3% traffic) → promote.
3. **Control must be team-owned and protected.** "Keep the control separate, untouched, owned by our team."
4. **Other teams just provide model + predictor.** Promo, NV may also provide a different shard. NV wants multiple score heads (7-day, same-day).
5. **GenAI carousels are bifurcated** — vertically boosted but NOT horizontally ranked by store ranker. VP value function needs to come into GenAI horizontal ranking. Proves the need for UBP.
6. **Code messiness makes experiments confusing.** "That code is really messy. There's so many overrides that I'm confused."
7. **YZ going on pat leave April 13** — get alignment before then.

---

## The Pivot

### Old POC thesis
> Vertical ranking is config-driven. An MLE drops a JSON file with step sequences, calibration entries, feature declarations, and the engine executes it.

### New POC thesis
> An MLE declares their experiment (model + layer + traffic %), and UBP handles traffic allocation within the layer and runs the experiment. No DV waterfall, no code changes, no HP engineer involvement.

### What stays (internal engine architecture)
- `Scorable` interface on domain types (no wrapper classes)
- `RankingStep<S>` / `RankingHandler` / `Ranker<S>` engine
- Phase 1: `RANK_ALL` wrapping entire legacy pipeline; Phase 2: granular step decomposition
- Auto-trace infrastructure (future)
- control.json as internal production behavior definition

### What changes (MLE-facing surface)
- **Contract simplifies**: experiment ID + layer + model name + traffic % + optional params
- **No MLE-authored pipeline configs**: MLEs don't write step sequences
- **Traffic management is #1 priority**: replaces the DV waterfall hell
- **control.json is internal**: HP team maintains it, MLEs don't touch it

### New priority breakdown
| Priority | Weight | Component |
|---|---|---|
| Per-layer traffic management | 40% | Replaces DV waterfall — declarative experiment allocation |
| Engine (internal) | 30% | Config-driven ranking with clean interfaces |
| MLE interface | 20% | Simple experiment declaration |
| Observability | 10% | Per-step tracing |

---

## Impact on Existing Documents

| Document | Impact | Action |
|---|---|---|
| `rfc.md` | Contract 1 over-scoped; missing traffic management | Simplify MLE config contract; add traffic management section |
| `plan.md` | Wrong priority order; missing traffic management | Reorder: traffic mgmt first, engine second |
| `northstar.md` | Vision correct, emphasis shifts | Add traffic management to "done" criteria |
| `mle-experiment-guide.md` | Over-complex MLE interface | Simplify all cases to model + layer + traffic % |
| `spec/impl/*` | Described FeedRow adapter design — never built | Archived to `spec/archive/impl/` (2026-03-23) |
| `feed-service feat/ubp-shadow-infra` | Config data classes over-engineered | Reassess; assembler still useful, config simplifies |

---

## Open Questions

1. **Traffic allocation mechanism**: UBP-owned composite DV per layer? Traffic router reading experiment registry? Automated waterfall?
2. **How does control protection work?** Separate runtime config that only HP team can modify?
3. **NV multi-score-head support**: How does the value function consume 7-day + same-day scores?
4. **GenAI carousel horizontal ranking**: When does VP value function get brought in?
5. **YZ's "online ranking experiments setup" doc**: Need to read it to understand current waterfall implementation details.
