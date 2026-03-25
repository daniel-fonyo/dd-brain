# Ads Blending in UBP: Industry-Standard Auction + ML Scoring Design

> How ads compete fairly with organic content for any carousel position via auctioning and ML scoring,
> within the chain-of-responsibility ranking pipeline.

---

## The Problem Today

The homepage has two completely separate ranking systems:

1. **Organic pipeline** — Universal Ranker (Sibyl) scores carousels/stores → vertical + horizontal ranking → sorted page
2. **Ads pipeline** — ICP auction scores ads → `HomepageAdsBlender` inserts ads into fixed slots **after** organic ranking

These systems are blind to each other. The result:

- **Ads land in fixed slots regardless of quality.** A low-relevance ad gets slot 0 in a top carousel while a better ad is buried in carousel 6 — because the auction doesn't know where carousels end up.
- **No displacement measurement.** When an ad takes a slot, nobody knows the organic store it replaced or whether the swap was value-positive.
- **Auction outcomes don't match placement.** High bidders get buried, low bidders get prime spots. Prices don't adjust. Merchant trust erodes.
- **VP tradeoffs are invisible.** The organic ranker optimizes for VP, but when an ad displaces an organic store, that VP loss is never evaluated.

---

## Industry Standard: Unified Auction with ML Scoring

Every major ads-in-feed system (Google, Meta, Amazon, LinkedIn) converges on the same pattern:

### 1. Common Value Currency

All candidates — organic and ad — express expected value on a **single comparable scale**:

```
EV(candidate, position) = pImp(position) × pAct(candidate) × vAct(candidate)
```

| Component | Organic | Ads |
|---|---|---|
| **pAct** | P(click\|impression) from Sibyl — calibrated CTR/CVR | P(click\|impression) from Ads ML model — calibrated pCTR |
| **vAct** | `gov_w × GOV + fiv_w × FIV + strategic_w × Strategic` | `bid_amount` (advertiser willingness to pay) + platform margin terms |
| **pImp** | Position decay `decay_rate^k` | Same position decay |

**Calibration is the linchpin.** If organic pCTR = 5% means "5 out of 100 users click" and ads pCTR = 5% means the same thing, then `pAct × vAct` is directly comparable. Without calibration, the scales are meaningless.

### 2. Generalized Second-Price (GSP) Auction

Industry standard for search/feed ads:

1. Each ad provides: `bid` (max willingness to pay per action) + `pCTR` (ML-predicted click probability)
2. **Rank by eCPM**: `eCPM = bid × pCTR` (expected cost per mille — same as `pAct × vAct` for ads)
3. **Price by GSP**: winner pays the minimum bid needed to maintain their position — `price = next_ad_eCPM / winner_pCTR + ε`
4. This incentivizes truthful bidding — advertisers don't overpay

Modern variants (Meta, Google) use VCG (Vickrey-Clarke-Groves) pricing for multi-slot allocation, which is more complex but provides stronger incentive compatibility.

### 3. Joint Organic-Ads Ranking

The key innovation: ads don't get inserted **after** ranking. They compete **during** ranking:

```
All candidates (organic + ads) → Score on common scale → Sort by EV → Apply constraints → Serve
```

**Constraints prevent ads from overwhelming the page:**
- Max ad load per carousel (e.g., ≤30% of slots)
- Min organic content guarantees (e.g., first 2 slots always organic)
- Quality floor (ad pCTR must exceed threshold to compete)
- Diversity rules (max N ads from same advertiser)
- Pacing (spread advertiser budget across time, don't exhaust in one session)

---

## How This Maps to UBP's Chain of Responsibility

The chain-of-responsibility pipeline with `RankingStep<S>` is the **exact right abstraction** for this. Ads blending becomes steps in the chain — not a parallel system bolted on after the fact.

### Current State (Phases 1-2): Organic Only

```
Pipeline: [MODEL_SCORING] → [MULTIPLIER_BOOST] → [DIVERSITY_RERANK] → [FIXED_PINNING]
Input: List<Scorable>  (organic carousels/stores only)
```

### Future State (Phase 3+): Unified Organic + Ads

```
Pipeline: [CANDIDATE_MERGE] → [CALIBRATION] → [VALUE_FUNCTION] → [AD_AUCTION] → [CONSTRAINT_ENFORCEMENT] → [DIVERSITY_RERANK] → [FIXED_PINNING]
Input: List<Scorable>  (organic + ad candidates on same interface)
```

### New Step Types for Ads

| Step | What It Does | Industry Pattern |
|---|---|---|
| `CANDIDATE_MERGE` | Combines organic candidates + ad candidates into one `List<Scorable>`. Ads team provides ad candidates via existing ICP retrieval (no change to auction/retrieval — only to blending). | Candidate pooling |
| `CALIBRATION` | Normalizes organic pAct and ads pAct to a common probability scale via isotonic regression or Platt scaling. HP-owned, runs offline calibration jobs. | Score calibration (Google, Meta) |
| `VALUE_FUNCTION` | Computes `EV = pAct × vAct` for every candidate. For organic: vAct from GOV/FIV/strategic weights. For ads: vAct from bid amount. | eCPM / expected value scoring |
| `AD_AUCTION` | Applies GSP/VCG pricing — determines what each ad actually pays if it wins its position. Records `opportunity_cost` = value of displaced organic. | Second-price auction |
| `CONSTRAINT_ENFORCEMENT` | Enforces ad load caps, quality floors, pacing budgets, min-organic guarantees. Demotes or filters ads that violate constraints. | Ad policy enforcement |

### Why Chain of Responsibility Works Perfectly Here

1. **Each step is independent and testable.** Calibration can be validated offline. The auction step can be unit-tested with mock candidates. Constraint enforcement can be tested against ad policy rules.

2. **Steps are composable and orderable.** The ads team might want `CALIBRATION → AUCTION → CONSTRAINTS`. An experiment might skip `DIVERSITY_RERANK`. The engine doesn't care — it runs whatever the step list says.

3. **Cross-cutting concerns are transparent.** The `StepHandler` wrapper automatically traces score deltas at every step. After `AD_AUCTION`, you can see exactly which organic store was displaced and the price paid. This is the opportunity cost logging that's impossible in the current separate-pipeline architecture.

4. **New ad types plug in via Scorable.** An ad carousel implements `Scorable` just like an organic carousel. The engine never does `if (item is AdCarousel)` — it ranks by EV, period. The ads team owns the `vAct` calculation (bid × margin adjustments); HP owns the engine.

5. **Horizontal and vertical ads use the same engine.** Carousel-level ads (which carousel gets an ad carousel) = vertical pipeline with ads. Slot-level ads (which position in a carousel gets an ad store) = horizontal pipeline with ads. Same `RankingStep` interface, different step type enum.

---

## Scorable Extension for Ads

Ads need additional metadata beyond what organic `Scorable` provides:

```kotlin
// Option A: Extend the interface
interface AuctionableScoreable : Scorable {
    val bidAmount: Double          // Advertiser's max bid
    val adQualityScore: Double     // Ad-specific pCTR from ads model
    val advertiserId: String       // For per-advertiser diversity constraints
    val campaignId: String         // For pacing budget tracking
    val isAd: Boolean              // Distinguishes ad from organic in constraints step
}

// Option B: Carry metadata in context (simpler, no interface change)
// Ad-specific data lives in a side map keyed by scorableId()
// Steps that need it (AUCTION, CONSTRAINTS) read from context
// Steps that don't (CALIBRATION, DIVERSITY) are unaware
```

**Option B is cleaner** for the chain-of-responsibility model. The core `Scorable` interface stays lean — `predictionScore` carries the unified EV after calibration + value function. Ad metadata lives in `RankingContext` as a side channel that only ad-aware steps access. Organic-only steps never know ads exist.

---

## Two-Dimensional Ads Blending (Preserving Current Capability)

The current `twoDimensionalSLBlend()` does:
1. **Vertical**: Which carousels are ad-eligible?
2. **Horizontal**: Within eligible carousels, which slots get ads?

In UBP, this naturally splits across the two pipeline layers:

| Dimension | UBP Layer | What Happens |
|---|---|---|
| Which **carousels** get ads? | Vertical `RankingPipeline` | Ad carousels compete with organic carousels for page positions. An `AdCarousel` implements `Scorable`, scored by EV. If its EV > an organic carousel's EV (after constraints), it wins the position. |
| Which **slots** in a carousel get ads? | Horizontal `RankingPipeline` | Ad stores compete with organic stores within a carousel. Same `Scorable` interface, same EV comparison, same constraint enforcement. |

This replaces the current cluster-based fixed-pattern blending (`[A1,O6],[A2,O6],[A1,O4]...`) with value-based competition. Higher-EV content wins, regardless of whether it's organic or an ad.

---

## Transition Path: Incremental, Not Big-Bang

### Phase 3a: Horizontal Ads Blending (within carousels)

**Why start here:** Within-carousel ads blending is simpler (one carousel at a time), has clear measurement (CVR/CTR per slot), and the current `GenericHorizontalBlenderV2` is the most constrained/problematic part of the current system.

1. Ad stores implement `Scorable` (or ad metadata carried in context)
2. `CALIBRATION` step normalizes organic store scores and ad scores
3. `VALUE_FUNCTION` computes EV for both
4. `AD_AUCTION` applies GSP pricing within carousel
5. `CONSTRAINT_ENFORCEMENT` applies quality floors, ad load caps, dedupe radius
6. Shadow-validate against `GenericHorizontalBlenderV2` output
7. Gradual rollout

### Phase 3b: Vertical Ads Blending (carousel-level)

1. Ad carousels (sponsored carousels) implement `Scorable`
2. Compete in vertical ranking pipeline against organic carousels
3. Additional constraint: max N ad carousels per page, min organic before first ad carousel
4. Shadow-validate against current parallel `HomepageAdsBlender` placement

### Phase 3c: Retire `HomepageAdsBlender`

Once both layers serve ads through the UBP pipeline, the parallel `HomepageAdsBlender` job is no longer needed. Its logic is subsumed by the constraint steps in each pipeline.

---

## Key Design Decisions

### 1. Who owns the auction?

**Ads team owns the auction logic; HP owns the engine.**

- Ads team implements `AdAuctionStep : RankingStep<S>` — they own bid validation, GSP pricing, pacing budget checks
- HP registers it in the step registry and controls when it runs in the pipeline
- This is exactly the partner self-service model from the RFC: "Partner teams implement their own `RankingStep` and HP registers it"

### 2. Where does calibration happen?

**HP owns calibration infrastructure; model teams own their models.**

- Calibration is a platform concern — it normalizes scores across content types
- Isotonic regression calibrator trained offline on logged (predicted score, actual outcome) pairs
- Each model team ensures their model is calibration-compatible (outputs probabilities)
- The `CALIBRATION` step applies the pre-trained calibrator at serving time

### 3. How do we handle latency?

**Ads retrieval is already parallel; auction is in-process and fast.**

- Current system: `HomepageAdsBlender` runs in parallel with post-processing. Ad candidates are already retrieved.
- UBP: ad candidates are merged into the organic list before the pipeline runs. The auction step is pure math (sort + price computation) — sub-millisecond.
- The only new latency is calibration lookup (pre-loaded in memory, O(log n) binary search) — negligible.

### 4. How do we preserve ghost ads / iROAS measurement?

Ghost ads (lift experiment control group) are critical for ads measurement.

- `CONSTRAINT_ENFORCEMENT` step maintains ghost ad tracking
- Ghost ads score normally but are flagged `isGhostAd = true`
- The step records ghost ad metadata on adjacent candidates (same as today)
- Serializer handles ghost ad output identically to current system

---

## What This Enables That's Impossible Today

| Capability | Today | With UBP Ads Blending |
|---|---|---|
| **Best ad in best position** | Ad placement decoupled from organic ranking | Ads compete for positions by EV — best ad gets best available position |
| **Displacement measurement** | No opportunity cost tracking | Every ad insertion logged with displaced organic value |
| **Dynamic ad load** | Fixed cluster patterns | Ad load adapts to content quality — more ads when organic is weak, fewer when it's strong |
| **Cross-type optimization** | Organic, ads, merch ranked independently | All content on one scale — the system shows whatever is highest value |
| **Position-aware pricing** | Auction doesn't know final position | GSP prices adjust for actual position value via pImp |
| **Experiment velocity** | Ads blending changes = code changes | New ad policy = new constraint config, no code deploy |

---

## Summary

The chain-of-responsibility pipeline with `Scorable` + `RankingStep<S>` is architecturally aligned with how every major platform does ads-in-feed. The key additions for ads:

1. **Calibration step** — normalizes all scores to a common scale (prerequisite)
2. **Value function step** — explicit `pAct × vAct` for all candidate types
3. **Auction step** — GSP/VCG pricing, owned by ads team
4. **Constraint step** — ad load caps, quality floors, pacing, ghost ads

These are all `RankingStep<S>` implementations. The engine is unchanged. The `Scorable` interface is unchanged. Ads become first-class citizens in the same pipeline, not a separate system bolted on afterward.

The northstar: **the engine doesn't know whether it's ranking an organic store or an ad. It just picks the highest-EV content for each position, subject to constraints.**
