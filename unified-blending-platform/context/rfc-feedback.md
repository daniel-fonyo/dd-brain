# RFC Feedback — Stakeholder Conversations

> Captures feedback from conversations with MLEs and engineers
> about UBP direction, pain points, and what they actually need.
> See also `pivot-analysis.md` for the strategic pivot this informed.

---

## Frank Zhang (NV Team BE Engineer) — March 2026

### #1 Pain: New Carousel Type Onboarding

No process exists for onboarding new carousel types to homepage ranking.

**Three stages (from YZ, Frank agrees but says it's "very ambiguous"):**
1. **Pin and get impressions** — UR has no context for the new format. Model doesn't break, but it can't give a good score. Need to feed it data first by guaranteeing impressions.
2. **Exploration via MAB** — Let UR rank, but with exploration bonus. UR still doesn't know it well, may rank it too low. MAB gives it a chance — if users don't click, lower it.
3. **Bake into model** — Retrain with collected impression/click data.

**Problems:**
- No defined thresholds: how many impressions before stage 2? When to enter stage 3?
- Cross-vertical ICE table onboarding is broken/not enforced — should be mandatory
- ~1 new carousel type per quarter, all handled ad hoc
- "Every new thing on the homepage, we need to ask: do you need to worry about vertical ranking?"

**What Frank wants:** Abstraction so a product team bringing a new carousel type doesn't need to manage ranking complexity. Currently it's either:
- "I need someone from core to partner closely" (expensive)
- "Fuck it, I'm just gonna pin it" (bad for ranking)

No middle ground exists.

### Coupling Pain Points

- Between horizontal ranking and vertical ranking
- Between HP and vertical landing page
- Vertical ranking experiments all under 1 parent DV

### Post-Processing Architecture (Frank's Mental Model)

```
1. Score carousel by model (Sibyl call)
   - Build feature sets, determine use case
   - Primary predictor: P(conv)
   - Additional predictors (parallel, independent):
     - Intent model
     - Exploration ranker model

2. Vertical blender layer
   - FIV (First Impression Value) compensation/boost
   - Long-term value that model can't predict (e.g., grocery conversion
     worth more than Rx long-term because customer becomes "super customer")

3. Whole page optimization
   - Diversification (window deduping — don't show McDonald's in 3 carousels)
   - Business/ops driven actions (e.g., "Tony says grocery goes to position 1")
```

### Key Quotes

- "People are just adding random things into [the post-processor], and I even see people adding PIN logic in serialization as well, which is even worse."
- "If we can abstract all these away, that'd be awesome."
- "I'm not from CORE, so I do not have enough time to work on it."
- "Every team has a different pain. When I come to the next person, it's not their pain."

---

## Yu Zhang (Manager) — March 2026

### #1 Pain: DV Waterfall for Per-Layer Traffic Management

See `context/pivot-analysis.md` for full details.

- 4 consumer cohorts → priority list → dummy DVs → reserve segments → waterfall
- "Right now, it's nasty work." Changing one experiment breaks all downstream.
- Wants: "I'm just telling you, hey, this experiment is taking that much traffic, I'm done."
- Expects Daniel to redesign the experiment traffic system.

### Other Key Points

- Vertical blending is the most mature work stream
- Start from what Daniel can handle, not MLE wishlist
- Frank/Harsha doing predictor splitting — incorporate if useful
- This is Daniel's own work, teams continue on old approach and switch over

---

## Dipali Ranjan (MLE) — March 2026

### #1 Pain: Model Experiment Workflow

See `context/pivot-analysis.md` for full details.

- Experiments = model swaps. "Predictor is done, you don't have to touch the predictor."
- Calibration workflow: model → calibrate → param tune (2-3% traffic) → promote
- Control must be team-owned and protected
- Other teams just provide model + predictor
- GenAI carousels bifurcated (vertically boosted, not horizontally ranked)
- Code is "really messy" with many overrides

---

## Synthesis: Each Stakeholder Has a Different #1 Pain

| Stakeholder | Role | #1 Pain |
|---|---|---|
| **Yu Zhang** | Manager | DV waterfall — per-layer experiment traffic management |
| **Dipali** | MLE | Model experiment lifecycle — config mess, ownership |
| **Frank** | NV BE | New carousel type onboarding — no abstraction, no process |

**What they all agree on:**
- The code is messy and poorly structured
- There's no abstraction layer — everything is coupled
- Adding anything new requires deep HP knowledge
- Experiments are confusing because of undocumented interactions

**How UBP addresses each pain:**

| Pain | UBP Component |
|---|---|
| DV waterfall | Per-layer traffic router with hash-based bucketing (future) |
| Model experiment config mess | Simplified experiment declaration (`mle-contract.md`) |
| New carousel type onboarding | `Scorable` interface — new type = add `override` + `withPredictionScore` |
| No step abstraction | `RankingStep` interface + step registry |
| No observability | Auto-trace per step via `MetricsHandler` |
| Code mess / coupling | `Ranker` engine: clean separation of concerns |

### The Incremental Path

Frank's pain point is the strongest argument for shipping **abstractions first, engine second**:

1. **`Scorable` interface** — Immediately useful. New carousel types implement one interface instead of touching 10+ files.
2. **`RankingStep` + `RankingHandler`** — Extract ranking logic into steps. Code gets cleaner AND we build toward the engine.
3. **`Ranker` engine** — Once interfaces exist, the engine is just a handler chain. Ship when abstractions are proven.
4. **Traffic splitting** — Future design effort, independent of engine.
