# Industry Research: Per-Layer Experiment Traffic Allocation in Ranking Systems

**Date**: 2026-03-19
**Purpose**: Inform UBP traffic management design by surveying how major tech companies handle mutually exclusive experiment traffic splitting within ranking layers.

---

## 1. The Foundational Model: Google's Overlapping Experiment Infrastructure

**Source**: Tang et al., KDD 2010 — the seminal paper that defined the industry standard.

### Core Architecture: Domains + Layers + Experiments

```
┌─────────────────────────────────────────────────────┐
│                    All Traffic                       │
├──────────────────┬──────────────────────────────────┤
│  Non-Overlapping │         Overlapping Domain        │
│     Domain       │                                   │
│  ┌────────────┐  │  ┌─────────┬─────────┬─────────┐ │
│  │ Exp A only │  │  │ UI Layer│ Search  │ Ads     │ │
│  │ OR         │  │  │         │ Results │ Layer   │ │
│  │ Exp B only │  │  │ Exp 1   │ Layer   │         │ │
│  │ OR         │  │  │   OR    │ Exp 3   │ Exp 5   │ │
│  │ Exp C only │  │  │ Exp 2   │   OR    │   OR    │ │
│  │            │  │  │         │ Exp 4   │ Exp 6   │ │
│  └────────────┘  │  └─────────┴─────────┴─────────┘ │
└──────────────────┴──────────────────────────────────┘
```

**Key rules**:
- **Within a layer**: mutually exclusive — a request is in at most ONE experiment per layer
- **Across layers**: overlapping — the same request can be in N experiments (one per layer)
- **Parameters are partitioned**: each parameter belongs to exactly one layer; no parameter can appear in multiple layers
- **Diversion is independent per layer**: different hash salts per layer ensure orthogonal assignment

### Traffic Diversion Mechanics
- Hash function `f(user_id, layer_salt)` produces deterministic bucket assignment
- Diversion types: user-id mod, cookie mod, cookie-day mod (rotating daily), random
- After diversion, **conditions** (country, language, browser, etc.) further filter eligibility

### Launch Layers (Rollouts)
- Separate from experiment layers, used for gradual feature rollout
- Priority order: `experiment layer value > launch layer default > system default`
- Prevents rollouts from interfering with running experiments

### Why This Matters for DoorDash UBP
Google's model maps directly: a "vertical ranking layer" and "horizontal ranking layer" are natural layer boundaries because they control different parameters. Experiments within each layer are mutually exclusive, but a user can simultaneously be in a vertical ranking experiment AND a horizontal ranking experiment.

---

## 2. Company-by-Company Survey

### 2.1 Microsoft / Bing

**System**: Bing Experimentation System (ExP platform, 60+ person team)

**Approach**: Multilayer design mirroring Google's architecture.
- Related experiments grouped into the same layer (e.g., ads text + ads background in "ads layer")
- Independent experiments run in separate layers in parallel
- Within a layer: mutual exclusion. Across layers: overlap.
- Runs 200+ experiments concurrently

**Key finding from Microsoft research**: Interaction effects between overlapping experiments occur in approximately **0.002% of cases** — virtually negligible. This is the most cited statistic in the industry for justifying overlapping designs.

### 2.2 Meta / Facebook

**System**: Gatekeeper (feature gating + traffic splitting), Quick Experiment, Airlock (mobile), Deltoid (analysis)

**Approach**: Tens of thousands of experiments running simultaneously. "No two people have the same Facebook."
- Gatekeeper permeates the entire stack (web, mobile, services)
- For ranking experiments (News Feed, Search): **shadow traffic forking** — fork traffic to two machine sets, throw away new results but compare data quality
- Mostly overlapping by default; isolation only when UX conflict is anticipated
- Strong interaction effects are rare per Facebook's experience; effects are "generally additive"

**Unique insight**: Facebook's culture gives engineers high flexibility, which sometimes leads to unintentional experiment abuse (cost of logging, interaction effects). They manage this through tooling and education rather than strict isolation.

### 2.3 Netflix

**System**: Custom XP platform, 150K-450K requests/sec

**Approach**: Overlapping by default with manual conflict avoidance.
- Users can participate in multiple A/B tests simultaneously
- Test metadata + allocation rules stored in Cassandra
- Conflict prevention: "ABlaze" visual tool provides test schedule view so owners avoid conflicts on the same page/element
- Assumes conflicts mostly occur between "close" features altering the same page/element

**Unique contribution**: **Return-aware experimentation** — models experimentation as a resource allocation problem. As thousands of experiments compete for finite user traffic, Netflix uses dynamic programming to optimize which experiments get traffic and when. Won Best Paper (Applied Data Science) at KDD 2025.

### 2.4 Spotify

**System**: Confidence (experimentation platform)

**Approach**: Hash-based **Bucket Reuse** with exclusivity tags.

**Technical details**:
- All users hashed into **1 million buckets** using `HASH(user_id, salt) % 1_000_000`
- Salt selected once globally; bucket map is fixed after that
- Random sampling at bucket level; random treatment allocation at user level within buckets
- **Exclusivity tags** on flag rules (e.g., `home-screen-shortcuts-ranker`) ensure all experiments for that surface are mutually exclusive

**The "Salt Machine"**:
- When buckets are freed after an experiment ends, a new salt reshuffles freed users into new buckets
- Prevents carryover effects from sequential experiments on the same buckets
- Uses a "compensation factor" to handle dilution (e.g., if 50% of buckets free, need 2x overallocation)

**Bucket Reuse migration**: Spotify moved to a single company-global bucket structure. This makes it trivial to coordinate experiments — two independently run programs can be merged, and a sample from a broken experiment can be quarantined for future experiments.

### 2.5 LinkedIn

**System**: XLNT platform

**Approach**: Fully overlapping and orthogonal by default with opt-in isolation.
- Experiments are overlapping by default
- Simple solutions for splitting traffic to run disjointly when needed
- Supports interaction analysis in full-factorial fashion
- Fractional factorial design for specific combinations
- Centralized service configuration independent of code release (via Rest.li + Databus)

### 2.6 Uber

**System**: XP platform (rewritten as "Project Citrus" in 2020)

**Approach**: Parameter-based with traffic splitter primitives.
- Experimentation exists as a **temporary override layer** on top of Flipr (config platform)
- `traffic_splitter` experiment controls a `traffic_slice` parameter
- Downstream experiments constrain on `traffic_slice == "A"` to get their slice
- Supports nested holdouts: uber-level > org-level > team-level
- Staged rollouts: small % → ramp → 100%
- Local evaluation (microsecond latency, not RPC)

### 2.7 DoorDash (Current State)

**Key systems**: Curie (analysis), Dash-AB (statistics engine), Display Modules (feed service)

**Approach**: Layered experiment framework + interleaving + MAB

**Parallelization (4X capacity improvement)**:
- Built layered experiment framework into Dynamic Values infrastructure
- Legacy system couldn't support multiple mutually exclusive experiments without ad-hoc implementations
- CUPED/CUPAC variance reduction: 10-20% sample size reduction
- Interleaving designs: 100x sensitivity vs. traditional A/B for ranking experiments

**Interleaving for ranking**: Instead of splitting users between algorithms, merge results from multiple algorithms and attribute user engagement. Traffic capped at 2-5% per interleaving experiment.

**Display Modules**: Generic content blocks powering homepage layout. Feed service assembles modules from backend, enabling flexible experimentation on layout, content, and ordering.

---

## 3. The "DV Waterfall" / Priority Chain Anti-Pattern

### What It Is
A pattern where experiments are evaluated in priority order, and each experiment "claims" traffic before the next one can. Lower-priority experiments only get leftover traffic.

```
User request arrives
  → Check Exp A (priority 1): if eligible, assign to A, STOP
  → Check Exp B (priority 2): if eligible, assign to B, STOP
  → Check Exp C (priority 3): if eligible, assign to C, STOP
  → Default behavior
```

### Why It's Problematic
1. **Traffic starvation**: Lower-priority experiments never get enough traffic for statistical significance
2. **Ordering bias**: The priority order itself introduces a systematic bias — users in Exp C are specifically those NOT eligible for A or B
3. **Scaling nightmare**: Adding a new experiment requires understanding and coordinating with all higher-priority experiments
4. **No parallelism**: Experiments cannot run independently; changing one affects all downstream
5. **Difficult to analyze**: The "leftover" population for each experiment is a biased subset

### How the Industry Avoids It

| Approach | How It Works | Who Uses It |
|----------|-------------|-------------|
| **Layer-based partitioning** | Parameters assigned to layers; within each layer, traffic is randomly split (not priority-ordered) | Google, Bing, DoorDash |
| **Hash-based bucketing** | Users deterministically assigned to buckets via hash; buckets allocated to experiments | Spotify, Optimizely, Amplitude |
| **Exclusion groups / slots** | Fixed slots created upfront; experiments assigned to non-overlapping slots | Optimizely, LaunchDarkly, AB Tasty |
| **Overlapping with monitoring** | Let experiments overlap; detect interaction effects automatically | Meta, Netflix, Statsig |
| **Traffic splitter primitive** | A dedicated "splitter" experiment carves traffic into named slices; downstream experiments constrain on their slice | Uber |

### The Correct Alternative: Random Partitioning Within a Layer

Instead of priority-ordered evaluation:

```
User request arrives
  → Hash(user_id, layer_salt) → bucket 0-999
  → Buckets 0-249: Exp A
  → Buckets 250-499: Exp B
  → Buckets 500-749: Exp C
  → Buckets 750-999: Default (holdout / no experiment)
```

Each experiment gets a **random, unbiased** sample. No priority ordering. Adding/removing an experiment only affects its bucket range.

---

## 4. N Experiments Competing Within One Layer

### The Core Problem
A ranking layer (e.g., vertical store ranking) might have 5 teams wanting to run experiments simultaneously, each needing 20% traffic for statistical power. That's 100% — no room for error, holdouts, or ramp-up.

### Standard Solutions

#### A. Fixed Bucket Allocation (Spotify / Optimizely Model)
- Pre-allocate N bucket ranges within the layer
- Each experiment gets a deterministic slice
- Simple, but rigid — unused capacity is wasted unless reallocated

#### B. Dynamic Allocation with MAB (DoorDash / Netflix Model)
- Multi-armed bandit dynamically shifts traffic toward winning variants
- Reduces opportunity cost of serving suboptimal variants
- Good for optimization; less clean for pure causal inference

#### C. Interleaving (DoorDash / Bing Model)
- For ranking experiments specifically: merge results from multiple algorithms within a single request
- 100x sensitivity improvement — need far less traffic per experiment
- Applicable when you're comparing ranked lists (perfect for vertical/horizontal ranking)
- DoorDash caps at 2-5% traffic per interleaving experiment

#### D. Sensitivity Boosting to Reduce Traffic Needs (DoorDash / Microsoft)
- CUPED/CUPAC variance reduction: 10-20% sample size reduction per experiment
- Triggered analysis: only measure users who actually encounter the change
- Surrogate metrics: faster-moving proxy metrics that require less traffic

#### E. Layered Parameter Partitioning (Google Model)
- If experiments modify different parameters, put them in different layers
- They can overlap (same user in both), effectively multiplying capacity
- Only works when parameters are truly independent

### Capacity Planning Formula (Simplified)
```
Available traffic per layer = Total eligible users
Experiments per layer = Available traffic / (min_sample_per_experiment × num_variants)
Boost factor = 1 / (1 - variance_reduction_from_CUPED)
Effective capacity = Experiments per layer × Boost factor
```

---

## 5. Hash-Based Bucketing Deep Dive

### The Standard Two-Step Process

**Step 1: Allocation** — Should this user be in the experiment at all?
```
allocation_hash = HASH(user_id, experiment_salt) % 10000
if allocation_hash < traffic_allocation_pct * 100:
    proceed to step 2
else:
    user is not in experiment (sees default)
```

**Step 2: Variant Assignment** — Which variant does this user see?
```
variant_hash = HASH(user_id, variant_salt) % 10000
if variant_hash < 5000:
    assign to control
else:
    assign to treatment
```

### Hash Function Selection Matters

| Hash Function | Uniform Distribution | Independence Between Experiments | Speed |
|--------------|---------------------|--------------------------------|-------|
| MD5 | Yes | Yes (no correlations) | Slower |
| SHA256 | Yes | Mostly (5-way interaction needed to see correlation) | Slower |
| MurmurHash3 | Yes | Yes (industry standard for experimentation) | Fast |
| SpookyHash | Yes | Yes (Yahoo switched to this after FNV issues) | Fast |
| FNV | Yes | **No** — introduces correlations | Fast |

**Critical lesson**: OfferUp and Yahoo both discovered that FNV hash produces correlated assignments across experiments. MurmurHash3 is the industry standard (used by Optimizely, Amplitude, most platforms).

### Salting for Independence
- Each layer/experiment uses a different salt
- `HASH(user_id + layer_salt)` ensures assignment in Layer A is independent of Layer B
- Without salting, a user in treatment for Exp A would always be in treatment for Exp B

### Bucket Reuse vs. Fresh Salts (Spotify's Insight)
- **Bucket Reuse**: One global salt, fixed bucket map. Experiments sample from buckets. Simple coordination.
- **Fresh Salts**: New salt per experiment. Better randomization but harder to coordinate exclusivity.
- Spotify chose Bucket Reuse for simplicity and coordination advantages.

---

## 6. Mapping to DoorDash Homepage Ranking (UBP Context)

### The DoorDash Surface Structure
```
Homepage
├── Vertical Ranking Layer (store feed ordering)
│   Parameters: vertical_rank_model, vertical_features, vertical_boost_weights
│
├── Horizontal Ranking Layer (within-carousel ordering)
│   Parameters: carousel_rank_model, carousel_features, item_scoring
│
├── Carousel Selection Layer (which carousels to show)
│   Parameters: carousel_eligibility, carousel_ordering, carousel_count
│
├── Layout/Blending Layer (intermixing carousels + stores)
│   Parameters: blend_ratio, layout_template, module_ordering
│
└── UI Layer (visual presentation)
    Parameters: card_design, carousel_style, pagination
```

### Recommended Architecture: Google-Style Layers with Spotify-Style Bucketing

#### Layer Definition
Each layer above controls independent parameters. A user can be in:
- 1 vertical ranking experiment
- 1 horizontal ranking experiment
- 1 carousel selection experiment
- 1 blending experiment
- 1 UI experiment
...all simultaneously. This is 5x the capacity of a single-layer system.

#### Within-Layer Traffic Splitting
Use hash-based bucketing (MurmurHash3):
```
bucket = MurmurHash3(user_id, layer_salt) % 1000

Layer: Vertical Ranking (1000 buckets)
├── Buckets 0-199:   Exp VR-1 (new ranking model A)
├── Buckets 200-399: Exp VR-2 (new ranking model B)
├── Buckets 400-449: Exp VR-3 (feature weight tuning)
├── Buckets 450-999: Default (production model)
└── [No priority chain — all assignments are random]
```

#### Handling the "N Experiments" Problem
For DoorDash's homepage ranking specifically:

1. **Separate layers for independent parameters** — vertical rank model vs. carousel selection vs. layout are different layers, can overlap
2. **Hash-based mutual exclusion within a layer** — multiple ranking model experiments within the vertical layer get random bucket slices
3. **Interleaving for ranking sensitivity** — when comparing ranking algorithms, use interleaving (100x sensitivity) to reduce traffic needs to 2-5%
4. **CUPED/CUPAC for all experiments** — 10-20% variance reduction across the board
5. **Default/holdout bucket** — always reserve a bucket range for production baseline (control)

#### Avoiding the DV Waterfall
The key principle: **experiments within a layer are peers, not a priority chain**. Traffic allocation is decided at experiment creation time by assigning bucket ranges. No experiment "outranks" another.

The evaluation logic should be:
```python
def get_experiment_assignment(user_id, layer):
    bucket = murmurhash3(user_id, layer.salt) % layer.num_buckets
    for experiment in layer.experiments:
        if bucket in experiment.bucket_range:
            return experiment.get_variant(user_id)
    return layer.default_value  # No experiment — use production default
```

NOT:
```python
# ANTI-PATTERN: Priority waterfall
def get_experiment_assignment_bad(user_id, layer):
    for experiment in sorted(layer.experiments, key=lambda e: e.priority):
        if experiment.is_eligible(user_id):
            return experiment.get_variant(user_id)
    return layer.default_value
```

---

## 7. Decision Framework: When to Isolate vs. Overlap

| Scenario | Approach | Rationale |
|----------|----------|-----------|
| Two experiments modify the same parameter (e.g., both change vertical rank model) | **Mutual exclusion** within a layer | Cannot apply two ranking models simultaneously |
| Two experiments modify different parameters (e.g., vertical ranking vs. carousel layout) | **Overlap** via separate layers | Independent parameters; no interaction |
| Experiment changes something that could break UX if combined with another | **Mutual exclusion** or careful monitoring | Rare but anticipated (e.g., red text + red background) |
| Need to measure long-term cumulative effect of all changes | **Holdout group** | Reserve 5-10% of traffic that sees no experiments |
| Ranking algorithm comparison | **Interleaving** within small traffic slice | 100x sensitivity; minimizes blast radius |
| Need maximum experiment velocity | **Overlap everything** + interaction monitoring | Industry consensus: interaction effects are ~0.002% |

---

## 8. Key Takeaways

1. **The industry consensus is: overlap by default, isolate selectively.** Google, Meta, Netflix, LinkedIn, Microsoft all run hundreds/thousands of overlapping experiments. Interaction effects are empirically ~0.002%.

2. **Layers are the standard abstraction.** Partition parameters into layers. Within a layer: mutual exclusion. Across layers: overlap. This is Google's model from 2010 and remains the gold standard.

3. **Hash-based bucketing (MurmurHash3) is the standard traffic splitting mechanism.** Deterministic, uniform, independent across layers when properly salted. Two-step process: allocation then variant assignment.

4. **The "waterfall" anti-pattern is solved by random bucket partitioning.** Experiments are peers within a layer, not a priority chain. Each gets a random, unbiased slice of traffic.

5. **For ranking experiments specifically, interleaving is a game-changer.** 100x sensitivity improvement means you need 1/100th the traffic. DoorDash already uses this.

6. **Sensitivity boosting (CUPED/CUPAC) effectively multiplies experiment capacity.** 10-20% variance reduction means 10-20% fewer users needed per experiment.

7. **Spotify's Bucket Reuse is the simplest coordination model.** One global bucket structure, experiments claim bucket ranges. Easy to coordinate exclusivity, merge programs, and quarantine bad samples.

---

## Sources

- [Google: Overlapping Experiment Infrastructure (KDD 2010)](https://research.google.com/pubs/archive/36500.pdf)
- [Google Patent: Overlapping Experiments (US8090703B1)](https://patents.google.com/patent/US8090703)
- [Spotify: New Experimentation Platform Part 1](https://engineering.atspotify.com/2020/10/spotifys-new-experimentation-platform-part-1)
- [Spotify: New Experimentation Platform Part 2](https://engineering.atspotify.com/2020/11/spotifys-new-experimentation-platform-part-2)
- [Spotify: Experimentation Coordination Strategy (Bucket Reuse)](https://engineering.atspotify.com/2021/03/spotifys-new-experimentation-coordination-strategy)
- [Spotify: Confidence Feature Flags](https://confidence.spotify.com/blog/feature-flags)
- [Netflix: It's All A/Bout Testing](https://netflixtechblog.com/its-all-a-bout-testing-the-netflix-experimentation-platform-4e1ca458c15)
- [Netflix: Return-Aware Experimentation](https://netflixtechblog.medium.com/return-aware-experimentation-3dd93c94b67a)
- [LinkedIn: XLNT A/B Testing Platform](https://engineering.linkedin.com/ab-testing/introduction-technical-paper-linkedins-ab-testing-platform)
- [LinkedIn: A/B Testing in Social Networks (PDF)](https://content.linkedin.com/content/dam/engineering/site-assets/pdfs/ABTestingSocialNetwork_share.pdf)
- [Uber: Under the Hood of XP](https://www.uber.com/blog/xp/)
- [Uber: Supercharging A/B Testing](https://www.uber.com/blog/supercharging-a-b-testing-at-uber/)
- [Microsoft: Online Experimentation at Microsoft (Stanford)](https://ai.stanford.edu/~ronnyk/ExPThinkWeek2009Public.pdf)
- [Microsoft Research: Experimentation Platform](https://www.microsoft.com/en-us/research/group/experimentation-platform-exp/)
- [DoorDash: Improving Experiment Capacity by 4X](https://careersatdoordash.com/blog/improving-experiment-capacity-by-4x/)
- [DoorDash: Interleaving Designs](https://careersatdoordash.com/blog/doordash-experimentation-with-interleaving-designs/)
- [DoorDash: MAB Platform](https://careersatdoordash.com/blog/experimentation-at-doordash-with-a-multi-armed-bandit-platform/)
- [DoorDash: Display Modules for Experimentation](https://careersatdoordash.com/blog/using-display-modules-to-enable-rapid-experimentation/)
- [DoorDash: Homepage Recommendation](https://careersatdoordash.com/blog/homepage-recommendation-with-exploitation-and-exploration/)
- [DoorDash: GenAI Homepage](https://careersatdoordash.com/blog/doordashs-next-generation-homepage-genai/)
- [Statsig: Embrace Overlapping A/B Tests](https://www.statsig.com/blog/embracing-overlapping-a-b-tests-and-the-danger-of-isolating-experiments)
- [Statsig: Interaction Effect Detection](https://www.statsig.com/blog/interaction-effect-detection)
- [Eppo: Mutual Exclusion (Layers) Docs](https://docs.geteppo.com/feature-flagging/concepts/mutual_exclusion/)
- [Eppo: Layers for Coordinated Experimentation](https://www.geteppo.com/blog/layers-enabling-coordinated-experimentation)
- [Eppo: When to Use Mutually Exclusive Experiments](https://www.geteppo.com/blog/when-should-i-use-mutually-exclusive-experiments)
- [Zhavzharov: Meta-Review of Overlapping A/B Test Practices](https://medium.com/@zhavzharovmikhail/interactions-in-overlapping-a-b-tests-meta-review-of-industry-practices-b4dd99ea75b8)
- [Towards Data Science: Assign Experiment Variants at Scale](https://towardsdatascience.com/assign-experiment-variants-at-scale-in-a-b-tests-e80fedb2779d/)
- [Depop: A/B Test Hash Bucketing](https://engineering.depop.com/a-b-test-bucketing-using-hashing-475c4ce5d07)
