[Draft] Unified Blending Platform
Author(s):
	Yu Zhang
	Status:
	Draft
	Created:
	Feb 26, 2026
	Last Updated:
	Mar 11, 2026
	Keywords:
	Ranking, blending, whole-page optimization, value function, experimentation, platform
	

Problem Statement
DoorDash's Homepage has evolved into a complex ecosystem serving multiple content types—Organic (Rx/NV), Ads, and Merchandising Platform—each optimized independently with fragmented systems, manual configurations, and heuristic-driven logic. While this approach enabled rapid iteration in early stages, we now face critical scaling bottlenecks that limit revenue growth, slow experimentation velocity, and create unsustainable engineering overhead.


The opportunity is clear: By unifying our blending infrastructure into a systematic, principled platform, we can unlock significant engineering efficiency gains, enable faster innovation across partner teams, and establish DoorDash as an industry leader in whole-page optimization—mirroring breakthroughs at companies like Pinterest, Meta, and Google.


________________


Theme 1: Technical Fragmentation & System Complexity
Impact: Unsustainable engineering cost, slow iteration speed, high error rates
1.1 Fragmented Optimization Without Global Coordination
System Layer: Ranking Engine, Blending Logic, Value Function


Problem: Multiple independent ranking systems (Organic UR, Ads ranker, VP ranker, Merchandising placement logic) optimize locally without awareness of each other, leading to suboptimal whole-page outcomes.


Concrete Example:


* Ads are inserted after all organic horizontal and vertical ranking completes
* The best-performing ad might be placed in the lowest-ranked carousel, missing impression opportunities
* Merchandising carousels use heuristic horizontal ranking and fixed vertical slots, operating entirely separately from organic blending logic


Stakeholder Impact:


* HP Team: Cannot reason about end-to-end page quality or make holistic trade-offs
* Ads Team: Revenue leakage due to poor ad positioning relative to organic quality
* Leadership: Missing revenue optimization opportunities across the entire page
1.2 DV Configuration Hell
System Layer: Configuration Management, Experimentation Framework


Problem: Manual DV setup with extensive backend engineering effort, nested references spread across multiple files, error-prone traffic splitting, and required cross-team coordination for every experiment.


Quantified Impact:


* ~2 weeks to set up a new multi-arm DV with proper traffic allocation
* 15+ files may need modification to insert new logic (value functions, boosting, filtering)
* High error rate: Nested DVs reference each other, making interaction effects hard to prevent
* Manual traffic splits cannot be validated until production deployment


Concrete Example:


discovery_p13n_hp_vertical_blending (DV A)
  ├─ references: discovery_p13n_vp_ranker_config (DV B)
  └─ nested in: discovery_p13n_merchandising_boost (DV C)
       └─ conflicts with: discovery_p13n_nv_program_boost (DV D)

Stakeholder Impact:


* HP Team: Excessive time spent on DV plumbing instead of ML innovation
* Partner Teams: Cannot launch experiments independently; blocked on HP eng availability
* Leadership: Slow experimentation velocity directly limits product iteration speed
1.3 Calibration Chaos: Incomparable Scoring Scales
System Layer: Value Function, Model Calibration, Score Normalization


Problem: Different content types ranked by separate models with incomparable score semantics, making it impossible to systematically compare and blend them.


Concrete Example:


* Organic carousels: UR scores (engagement-based)
* Ads: eCPM or bid-weighted pCTR
* Merchandising: MAB-based placement scores
* VP stores: Predicted profit (pConv × pDasherCost × pCommission)


Without calibration and unified value functions, blending decisions rely on manually tuned multipliers inserted at different stages, with no principled trade-off analysis.


Stakeholder Impact:


* HP Team: Cannot systematically A/B test blending strategies
* Partner Teams: Model improvements don't translate to better placement without manual retuning
* Leadership: No visibility into opportunity costs of placement decisions
1.4 Observability Gaps: No Stage-Wise Logging
System Layer: Logging Infrastructure, Observability, Metrics


Problem: Lack of detailed logging at different ranking stages (retrieval → ranking → blending → filtering) makes debugging nearly impossible and requires manual code changes for every new metric.


Quantified Impact:


* Days to add new logging for experiment analysis
* No systematic way to compute counterfactuals (what would have ranked differently?)
* Cannot measure value leakage at each stage of the pipeline


Stakeholder Impact:


* HP Team: Blind to where ranking quality degrades in the pipeline
* Partner Teams: Cannot validate that their models are being used correctly in production
* Leadership: Limited visibility into system health and performance


________________


Theme 2: Value Function & Ranking Quality Gaps
Impact: Suboptimal business outcomes, missed revenue opportunities, poor user experience
2.1 No Systematic Value Function for Business Goals
System Layer: Value Function Framework, Business Objective Optimization


Problem: No principled value function to achieve business objectives (revenue, GOV, retention). Instead, teams rely on ablations and manual tuning without ongoing impact measurement.


Concrete Examples:


* Inflation deduplication: Uses ablation logic instead of value-based trade-offs
* Store quality deduplication: Delayed due to lack of systematic framework
* Quality-related projects cannot be prioritized without principled value measurement


Stakeholder Impact:


* HP Team: Cannot make data-driven decisions on quality vs. quantity trade-offs
* Partner Teams: No clear signal on how to optimize their content for HP placement
* Leadership: Business goals (e.g., "improve quality") lack quantifiable optimization targets
2.2 Legacy Heuristics Accumulate Without Measurement
System Layer: Ranking Logic, Technical Debt, Code Quality


Problem: Heuristics applied as quick fixes become permanent legacy code without continuous monitoring or improvement.


Concrete Examples:


* Inflation multiplier: Hardcoded adjustment factor applied in ranking, never revisited
* NV program boosting: Multiplier inserted in 3+ different places across ranking pipeline
* No systematic A/B testing or offline simulation to validate heuristic impact over time


Quantified Impact:


* 50+ hardcoded multipliers/thresholds scattered across codebase
* Unknown impact: No measurement of whether these heuristics still provide value
* Technical debt: Each heuristic increases system complexity and maintenance cost


Stakeholder Impact:


* HP Team: Increasingly complex codebase that's difficult to reason about
* Partner Teams: Unclear how to propose changes without breaking existing heuristics
* Leadership: Risk of degraded user experience from unmaintained, outdated logic
2.3 Suboptimal Ads Insertion (Post-Organic Ranking)
System Layer: Blending Engine, Ads Integration, Joint Optimization


Problem: Ads are inserted after organic horizontal and vertical ranking completes, preventing true joint optimization and causing inefficiency.


Concrete Example:


Current Flow:
1. Rank organic carousels vertically (positions 1-10)
2. Rank stores horizontally within each carousel
3. Insert ads into fixed slots or append ad carousels


Problem: The highest-value ad might belong in position 2,
but it's forced into position 8 because organic ranking
already claimed positions 1-7.

Quantified Impact:


* Estimated 10-20% revenue leakage from suboptimal ad positioning
* Cannot leverage organic quality signals to inform ad placement during ranking


Stakeholder Impact:


* Ads Team: Revenue loss due to poor ad placement relative to organic items
* HP Team: Cannot optimize globally; forced to "patch" ads into pre-ranked organic lists
* Leadership: Significant missed monetization opportunity
2.4 No Cold-Start Strategy for New Content
System Layer: Exploration, Cold-Start Handling, Pacing Controls


Problem: No systematic approach for cold-starting new verticals (NV Mx) or new content types. Current solution relies on pinning without exposure control, pacing, or ranking.


Concrete Example:


* New NV vertical launches: Pinned to position 2 for all users for 2 weeks
* No gradual ramp-up or quality-based exposure control
* No measurement of opportunity cost (what organic content was displaced?)


Stakeholder Impact:


* Partner Teams (NV, Merch): Cannot launch new content confidently without disrupting UX
* HP Team: No systematic platform capability to offer; forced to manual pinning
* Leadership: Slow time-to-market for new product initiatives
2.5 Opportunity Cost Blindness for Boosting/Pinning
System Layer: Value Function, Counterfactual Analysis, Business Intelligence


Problem: When content is boosted or pinned for business reasons, we don't measure the opportunity cost (what was displaced and what value was lost).


Quantified Impact:


* Unknown trade-offs: No visibility into GOV, engagement, or revenue loss from boosting decisions
* Cannot answer: "Is boosting NV trial worth the organic engagement we're sacrificing?"


Stakeholder Impact:


* HP Team: Cannot provide data-driven recommendations on boosting strategies
* Partner Teams: Unclear whether their boosting requests are helping or hurting overall metrics
* Leadership: Making strategic decisions without full visibility into costs
2.6 Cannot Support Fine-Tuning for Cohorts and Contexts
System Layer: Personalization, Context-Aware Ranking, Segmentation


Problem: Hard to personalize blending decisions for different geographic areas, consumer cohorts (new vs. existing users), or contexts (browse vs. search).


Concrete Example:


* Same blending logic applied to all users regardless of order history, preferences, or context
* Cannot optimize differently for high-frequency vs. low-frequency users
* No geo-specific tuning despite vastly different supply/demand dynamics


Stakeholder Impact:


* HP Team: Limited ability to deliver personalized experiences at scale
* Partner Teams: Cannot leverage their domain expertise to define cohort-specific strategies
* Leadership: Missing personalization opportunities that drive engagement and retention


________________


Theme 3: Engineering Velocity & Organizational Inefficiency
Impact: Slow product iteration, blocked partner teams, duplicated work
3.1 Experimentation Bottleneck
System Layer: Experimentation Framework, Configuration Management, Dev Velocity


Problem: Launching a new blending experiment requires extensive cross-team coordination, manual DV setup, and backend code changes. This creates a critical bottleneck that slows product iteration.


Quantified Impact:


* 2-3 weeks from experiment idea to production launch
* Requires sync across 3+ teams (HP, NV, Ads, Merch Platform)
* HP team acts as gatekeeper, blocking partner teams from iterating independently


Stakeholder Impact:


* HP Team: Overwhelmed with experimentation requests; cannot focus on platform innovation
* Partner Teams: Innovation velocity constrained by HP's bandwidth
* Leadership: Slow experimentation directly impacts competitive positioning
3.2 Ownership Ambiguity (Everyone Does a Little of Everything)
System Layer: Platform Architecture, Team Ownership, API Contracts


Problem: No clear boundaries between platform (HP) and partner teams. Everyone contributes a little to every layer, resulting in inefficiency and lack of accountability.


Concrete Example:


Who owns organic ranking calibration? HP + NV both involved
Who owns ads blending logic? HP + Ads both contribute
Who owns merchandising placement scoring? HP + Merch Platform unclear

Stakeholder Impact:


* HP Team: Cannot build deep platform expertise; constantly context-switching
* Partner Teams: Unclear what they own vs. what HP provides
* Leadership: Difficult to hold teams accountable for outcomes
3.3 Merchandising Platform Reinventing Wheels
System Layer: Platform Reusability, Code Duplication, Cross-Org Efficiency


Problem: Merchandising Platform is building separate horizontal and vertical blending logic using heuristics and fixed slots, duplicating work that HP team has already invested in.


Quantified Impact:


* 3-6 months of duplicated engineering effort
* Likely to produce suboptimal results compared to ML-driven blending
* Will face the same scaling problems HP encountered (fragmentation, DV hell, etc.)


Stakeholder Impact:


* Merch Platform Team: Wasting engineering resources on infrastructure instead of product features
* HP Team: Missing opportunity to consolidate and own blending platform
* Leadership: Inefficient resource allocation across org
3.4 Domain Expertise Bottleneck
System Layer: Platform Extensibility, Partner Integration, Contribution Model


Problem: HP team cannot be experts in every vertical (Rx, NV, Ads, Merch, etc.). Current architecture forces HP to deeply understand domain logic to incorporate it into ranking, creating a bottleneck.


Concrete Example:


* NV team wants to test a new trial boosting strategy → Requires HP eng to understand NV business logic and modify ranking code
* Ads team improves bidding models → Requires HP to integrate new signals manually
* Merchandising wants promo-aware ranking → HP becomes the blocker


Stakeholder Impact:


* HP Team: Cannot scale; becomes bottleneck for all partner teams
* Partner Teams: Cannot leverage their domain expertise without depending on HP
* Leadership: Slows down org-wide innovation velocity


________________


Summary: The Case for Urgent Action
The current fragmented state creates a compounding cost:


* Engineering Cost: 50-60% of HP eng time spent on DV plumbing, manual tuning, and firefighting
* Opportunity Cost: Estimated 10-20% revenue leakage from suboptimal blending
* Innovation Cost: 2-3 week experimentation cycle vs. industry leaders achieving daily iteration


Meanwhile, the opportunity is clear:


* Industry leaders (Pinterest, Meta, Google) have demonstrated 15-30% efficiency gains through unified blending platforms
* Partner teams (NV, Ads, Merch Platform) are ready to contribute and invest in shared infrastructure
* DoorDash is uniquely positioned to build a multi-business blending platform (Rx + NV + Ads + Merch) that spans the entire discovery journey


The Unified Blending Platform is not just an incremental improvement—it's a strategic investment that will fundamentally transform how DoorDash optimizes content discovery and monetization at scale.


________________


Vision & Principles
North Star Vision
The Unified Blending Platform aims to transform DoorDash into an industry leader in whole-page optimization by establishing a principled, ML-driven platform that:


1. Enables Whole-Page Optimization Across All Content Types - Seamlessly blends Organic (Rx + NV), Ads, and Merchandising content through unified value functions, replacing fragmented local optimization with global page-level optimization that maximizes user value and business objectives simultaneously.


2. Replaces Heuristics with Principled Value-Based Ranking - Establishes systematic value functions (pImp × pAct × vAct) that replace 50+ hardcoded heuristics with data-driven, measurable optimization. Every ranking decision becomes explainable through calibrated models and constrained optimization, enabling transparent trade-off analysis between user experience, revenue, and long-term value.


3. Empowers Self-Service Experimentation - Transforms experimentation from a 2-3 week cross-team coordination process into a 1-2 day self-service workflow. Partner teams (NV, Ads, Merchandising Platform) can independently launch experiments, contribute domain-specific models, and iterate on their content strategies without depending on HP team as a bottleneck.


4. Delivers Real-Time Adaptive Blending - Evolves from static, rule-based placement to ML-driven, context-aware optimization that adapts to user behavior, session intent, market dynamics, and business constraints in real-time. Enables personalization at scale across geographic areas, consumer cohorts, and discovery contexts.


5. Eliminates Manual Configuration Overhead - Abolishes DV configuration hell through declarative configuration, automated calibration, and composable experimentation framework. Engineers focus on ML innovation instead of configuration plumbing.


Long-Term Aspiration: Establish DoorDash's Unified Blending Platform as the gold standard for multi-business discovery optimization, positioning the company alongside industry leaders like Pinterest (RL-based utility tuning), Meta (whole-page optimization), and Google (two-tower ranking architectures).


________________


Core Design Principles
These principles guide all technical decisions and architecture choices:
1. Platform-First Mindset
Philosophy: HP team owns reusable, extensible blending infrastructure; partner teams contribute domain-specific models, value functions, and business logic.


In Practice:


* HP owns the blending engine, value function framework, calibration infrastructure, and experimentation platform
* Partner teams own candidate generation, domain-specific ranking models, and content-specific value models
* Clear API boundaries enable independent iteration while maintaining platform consistency


Trade-Off: Requires upfront platform investment but unlocks exponential long-term efficiency gains
2. Separation of Concerns
Philosophy: Clean architectural boundaries between retrieval, ranking, blending, filtering, and serving layers.


In Practice:


* Retrieval Layer: Partner teams provide candidates with standardized schema
* Ranking Layer: Domain-specific rankers (Organic UR, Ads ranker, Merchandising placement) produce calibrated scores
* Blending Layer: HP platform applies unified value function and optimization
* Filtering Layer: Apply constraints (diversity, quality, business rules) post-blending
* Serving Layer: Format and deliver ranked results to clients


Trade-Off: More architectural complexity but enables independent testing, debugging, and optimization at each layer
3. Unified Value Representation
Philosophy: All content types (Organic, Ads, Merchandising) must be expressible in a common value function to enable fair comparison and principled blending.


Core Formula:


Expected Value = pImp × pAct × vAct


Where:
- pImp = Probability of Impression (position-dependent)
- pAct = Probability of Action (CTR, CVR, etc.)
- vAct = Value of Action (GOV, revenue, LTV, strategic value)

In Practice:


* All content types calibrated to comparable scales
* Value function components computed consistently across content types
* Blending decisions based on expected value ordering with configurable constraints


Trade-Off: Requires significant calibration effort but provides mathematically sound foundation for optimization
4. Experimentation by Default
Philosophy: Every ranking, blending, and filtering decision must be experiment-configurable with A/B testing built-in from day one.


In Practice:


* Declarative configuration for all parameters (weights, thresholds, constraints)
* Automatic traffic splitting and bucketing
* Stage-wise logging for counterfactual analysis
* Offline simulation capability before production deployment


Trade-Off: Additional infrastructure complexity but dramatically accelerates learning velocity
5. Mathematical Rigor
Philosophy: Frame blending as a constrained optimization problem with explicit objective functions and measurable trade-offs.


Core Optimization:


Maximize: Σ (pImp_i × pAct_i × vAct_i)


Subject to:
- Quality constraints (minimum engagement threshold)
- Diversity constraints (content type distribution)
- Business constraints (ad load, pacing budgets)
- Fairness constraints (exposure guarantees)

In Practice:


* Use Lagrangian relaxation to derive blending value functions
* Tune multipliers (λ parameters) through offline optimization and online A/B testing
* Provide transparent reasoning for every placement decision


Trade-Off: Steeper learning curve but enables principled decision-making and systematic optimization
6. Progressive Enhancement
Philosophy: Start with simple baselines (V0), validate through experiments, then add sophistication incrementally based on proven value.


Evolution Path:


* V0: Basic calibration + simple blending (weeks 1-8)
* V1: Add diversity, constraints, cold-start handling (weeks 9-16)
* V2: Context-aware personalization, advanced value functions (weeks 17-24)
* V3+: RL-based optimization, multi-surface coordination (months 7+)


In Practice:


* Ship fast, learn fast, iterate based on data
* Never block V1 launch on V2 features
* Modular architecture enables parallel development


Trade-Off: Longer time to "perfect" system but dramatically reduces execution risk
7. Observability & Transparency
Philosophy: Detailed stage-wise logging, counterfactual analysis, and explainability are first-class platform features, not afterthoughts.


In Practice:


* Log scores, features, and decisions at every pipeline stage (retrieval → ranking → blending → filtering)
* Compute counterfactuals: "What would have ranked differently with treatment X?"
* Provide experiment dashboards with automatic metric computation
* Enable partner teams to validate that their models are used correctly


Trade-Off: Higher logging costs but critical for debugging, optimization, and trust
8. Fail-Safe & Graceful Degradation
Philosophy: System defaults to safe, proven baseline if advanced features fail. Never compromise user experience due to platform issues.


In Practice:


* Fallback to simple blending if value function computation times out
* Degrade to position-based ranking if calibration service unavailable
* Circuit breakers for each advanced feature with automatic rollback
* Comprehensive monitoring and alerting for all platform components


Trade-Off: Additional defensive code but protects user experience and builds stakeholder trust


________________


Success Criteria
The platform will be considered successful when it achieves:
Engineering Efficiency Metrics
1. Experiment Launch Velocity


   * Target: Reduce experiment launch time from 2-3 weeks → 3-5 days
   * Measurement: Time from experiment proposal to production deployment
   * Milestone: Achieve target for 70% of experiments by V1 launch; 85% by V2


2. Platform Adoption


   * Target: 2+ partner teams (NV, Ads, or Merchandising Platform) actively using platform APIs
   * Measurement: Number of experiments launched via platform per quarter
   * Milestone: 1-2 partner teams launch at least 2 experiments independently per quarter by V1; scale to 3+ teams by V2


3. Code Quality & Technical Debt Reduction


   * Target: Eliminate 30+ hardcoded heuristics, reduce codebase complexity by 20%
   * Measurement: Cyclomatic complexity, number of hardcoded multipliers/thresholds, lines of code
   * Milestone: Achieve 30% reduction in technical debt by V2 completion


4. Experimentation Throughput


   * Target: 3-5x increase in experimentation throughput (experiments per quarter)
   * Measurement: Total experiments launched by HP + partner teams
   * Milestone: Achieve 3x by V1 launch; 5x by end of year 1
User Experience Protection
5. Engagement Metrics Guardrails
   * Target: Maintain or improve engagement metrics (CTR, CVR, retention) throughout migration
   * Measurement: Homepage CTR, session CVR, D7/D30 retention
   * Guardrail: No statistically significant regression in any core metric during platform rollout


________________


Guiding Philosophy
"Start simple, iterate fast, optimize systematically."


The Unified Blending Platform prioritizes:


* Speed to value over perfection
* Data-driven decisions over intuition
* Platform leverage over point solutions
* Partner empowerment over centralized control
* Long-term sustainability over short-term hacks


By adhering to these principles, we build a foundation that scales with DoorDash's growing complexity while accelerating—not hindering—innovation velocity across the entire discovery organization.


________________


Solution Overview
The Unified Blending Platform introduces a systematic, ML-driven framework that replaces fragmented, heuristic-based ranking with principled whole-page optimization. The platform establishes clear separation of concerns: HP team owns the blending infrastructure, while partner teams (NV, Ads, Merchandising Platform) contribute domain-specific models and value functions through well-defined APIs.


________________


Core Architecture: Two-Layer Blending System
The platform operates on a two-layer blending architecture that optimizes content placement at both horizontal and vertical dimensions:


┌─────────────────────────────────────────────────────────────┐
│                    Homepage (User View)                      │
├─────────────────────────────────────────────────────────────┤
│  Position 1: [Organic Rx Carousel]                          │
│              [Store A] [Store B] [Store C] [Store D] ...    │ ← Layer 1: Horizontal Blending
│                                                              │
│  Position 2: [Ads Carousel]                                 │
│              [Ad 1] [Ad 2] [Ad 3] ...                       │ ← Layer 1: Horizontal Blending
│                                                              │
│  Position 3: [Organic NV Carousel]                          │
│              [Store E] [Store F] [Store G] ...              │ ← Layer 1: Horizontal Blending
│                      ↑                                       │
│  Position 4: [Merchandising Spotlight]                      │   Layer 2: Vertical Blending
│              [Promo Content] ...                            │   (Carousel-level optimization)
│                      ↓                                       │
│  Position 5: [Organic Rx Carousel]                          │
│              [Store H] [Store I] ...                        │ ← Layer 1: Horizontal Blending
│                                                              │
└─────────────────────────────────────────────────────────────┘

Layer 1: Horizontal Blending (Store-Level within Carousels)


* Ranks individual items (stores, ads, products) within each carousel
* Optimizes for expected value at the item level
* Currently active for VP profit optimization (stores), will extend to Ads and Merchandising items


Layer 2: Vertical Blending (Carousel-Level across Homepage)


* Ranks entire carousels (Organic Rx, Organic NV, Ads, Merchandising) across vertical positions
* Optimizes for whole-page expected value while maintaining diversity and quality constraints
* Replaces current fragmented vertical ranking with unified value-based blending


Key Innovation: Unlike current systems where Ads are inserted after organic ranking completes, the Unified Blending Platform enables simultaneous optimization across both layers, allowing high-value ads to compete fairly with high-value organic content at both carousel and item levels.


________________


Unified Value Function Framework
All content types—Organic (Rx/NV), Ads, and Merchandising—are evaluated using a common value function that enables fair comparison and principled blending:


Expected Value = pImp × pAct × vAct


Where:
┌─────────────────────────────────────────────────────────────┐
│ pImp (Probability of Impression)                            │
│   - Position-dependent: decreases as position increases     │
│   - Computed from historical data or NDCG-like decay        │
│   - Personalized by consumer cohort, context, device        │
└─────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────┐
│ pAct (Probability of Action)                                │
│   - Click probability (pCTR)                                │
│   - Conversion probability (pCVR)                           │
│   - Relevance probability (pRelevance) for search contexts  │
│   - Calibrated across all content types to comparable scale │
└─────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────┐
│ vAct (Value of Action)                                      │
│   Organic: GOV per order + long-term value (LTV, trials)   │
│   Ads: Ad revenue (bid × pCTR) + opportunity cost          │
│   Merchandising: Promo value + strategic value (awareness)  │
│   - All values normalized to common currency (USD)         │
└─────────────────────────────────────────────────────────────┘

Calibration is Critical: The platform provides centralized calibration infrastructure (HP-owned) that ensures pAct predictions from different models (Organic UR, Ads ranker, Merchandising MAB) are comparable and can be fairly blended.


________________


Constrained Optimization for Blending
The blending engine frames content placement as a constrained optimization problem, moving beyond simple score-based ranking to principled trade-off analysis:


Objective:
Maximize: Σ (pImp_i × pAct_i × vAct_i)
          i ∈ all_candidates


Subject to:
┌─────────────────────────────────────────────────────────────┐
│ Quality Constraints                                         │
│   - Minimum engagement threshold (e.g., CVR ≥ 2.5%)        │
│   - Prevents low-quality content from ranking high          │
└─────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────┐
│ Diversity Constraints                                       │
│   - Content type distribution (e.g., max 30% ads)           │
│   - Vertical diversity (e.g., Rx vs NV balance)             │
│   - Category/brand diversity within carousels               │
└─────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────┐
│ Business Constraints                                        │
│   - Ad load caps and pacing budgets                         │
│   - Strategic boosting/pinning with opportunity cost        │
│   - Cold-start exposure guarantees for new content          │
└─────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────┐
│ Fairness Constraints                                        │
│   - Minimum exposure for all content types                  │
│   - Anti-staleness (rotate content, avoid blindness)        │
└─────────────────────────────────────────────────────────────┘

Solution Approach: The platform uses Lagrangian relaxation to convert the constrained optimization problem into an unconstrained formulation with learned multipliers (λ parameters), enabling efficient online blending algorithms:


Mathematical Formulation:


The constrained optimization problem:


Maximize: Σ (pImp_i × pAct_i × vAct_i)
Subject to: g_j(ranking) ≥ threshold_j  for all constraints j

Is converted via Lagrangian relaxation to:


L(ranking, λ) = Σ (pImp_i × pAct_i × vAct_i) + Σ λ_j × (g_j(ranking) - threshold_j)

Where λ_j are Lagrange multipliers representing the marginal value of relaxing constraint j.


Key Insight from Industry Practice (inspired by Pinterest's utility tuning, Google's LambdaRank, and Uber's marketplace optimization):


Instead of solving the complex constrained optimization online, we:


1. Offline: Learn optimal λ parameters from historical data using:
   * Gradient-based optimization on replay logs
   * Counterfactual policy evaluation
   * Grid search with business metric validation
2. Online: Use learned λ to compute adjusted value function:


Adjusted_Value_i = pImp_i × pAct_i × vAct_i
                   + λ_quality × quality_boost_i
                   + λ_diversity × diversity_penalty_i
                   - λ_ad_load × ad_load_penalty_i

3. Greedy Selection: Rank by Adjusted_Value with provable approximation guarantees


Practical Implementation:


* Start with manual λ tuning based on business requirements (V0)
* Transition to offline learned λ using historical optimization (V1)
* Evolve to online adaptive λ using reinforcement learning (V2+)


This approach enables:


* Efficiency: O(N log N) greedy blending instead of exponential constraint satisfaction
* Interpretability: Each λ parameter has clear business meaning
* Tunability: Easy to adjust trade-offs via λ without code changes
* Provable guarantees: Lagrangian duality provides bounds on optimality gap


________________


Platform Components & Ownership Model
The platform establishes clear ownership boundaries between HP (platform) and partner teams (domain logic):
HP Team Owns (Platform Infrastructure)
1. Blending Engine


   * Two-layer blending logic (horizontal + vertical)
   * Constraint satisfaction and optimization algorithms
   * Greedy merge, diversity reranking, slot allocation


2. Value Function Framework


   * pImp computation and personalization
   * Unified value function application (pImp × pAct × vAct)
   * Value normalization across content types


3. Calibration Infrastructure


   * Isotonic regression for pAct calibration
   * Cross-content-type calibration validation
   * Calibration monitoring and drift detection


4. Experimentation Platform


   * Declarative configuration DSL
   * Automatic traffic splitting and bucketing
   * Stage-wise logging and counterfactual analysis
   * Experiment dashboard and metric computation


5. Observability & Tooling


   * Detailed pipeline logging (retrieval → ranking → blending → filtering)
   * Opportunity cost computation
   * Explainability tools (why did content rank at position X?)
   * Offline simulation and replay
Partner Teams Own (Domain Logic)
1. Candidate Generation & Retrieval


   * NV team: Organic NV candidate retrieval and filtering
   * Ads team: Ads candidate generation and bidding
   * Merchandising Platform: Placement content and promo logic


2. Domain-Specific Ranking Models


   * NV team: Organic ranker improvements (UR, MoE, etc.)
   * Ads team: Ads ranking models and pacing logic
   * Merchandising Platform: Placement scoring and MAB


3. Content-Specific Value Models


   * Each team defines vAct for their content type
   * NV team: GOV models, LTV models (trials, new verticals)
   * Ads team: Revenue models, opportunity cost models
   * Merchandising Platform: Promo value, awareness value


4. Experimentation & Iteration


   * Launch experiments independently via platform APIs
   * Contribute new constraints or objectives via config
   * Monitor partner-specific metrics and dashboards


________________


How the Platform Addresses Key Pain Points
The Unified Blending Platform systematically resolves the 14 pain points identified in the Problem Statement:


Pain Point Category
	How Platform Addresses It
	Fragmented Optimization
	Unified value function enables fair comparison across Organic, Ads, Merchandising; simultaneous two-layer optimization
	DV Configuration Hell
	Declarative config DSL + automatic traffic splitting reduces setup time from 2-3 weeks to 3-5 days
	Calibration Chaos
	Centralized calibration infrastructure ensures all pAct predictions are comparable; HP owns calibration validation
	Observability Gaps
	Stage-wise logging + counterfactual analysis built-in; no manual code changes for new metrics
	No Systematic Value Function
	pImp × pAct × vAct framework replaces ablations and heuristics; all decisions explainable and measurable
	Legacy Heuristics
	30+ heuristics eliminated; remaining logic converted to configurable constraints in optimization framework
	Suboptimal Ads Insertion
	Ads compete fairly at both layers (horizontal + vertical) during blending, not inserted post-ranking
	No Cold-Start Strategy
	MAB + pacing controls enable systematic cold-start with exposure guarantees and opportunity cost tracking
	Opportunity Cost Blindness
	Counterfactual logging measures what was displaced by boosting/pinning; transparent trade-off visibility
	Cannot Fine-Tune for Cohorts
	Personalized pImp, context-aware constraints, and segmented value functions enable cohort-specific optimization
	Experimentation Bottleneck
	Self-service config + platform APIs enable partner teams to launch experiments independently
	Ownership Ambiguity
	Clear API boundaries: HP owns blending infrastructure; partners own domain models and value functions
	Merchandising Reinventing Wheels
	Merchandising Platform adopts HP's two-layer blending instead of building separate logic
	Domain Expertise Bottleneck
	Plugin architecture allows partners to contribute models/constraints without HP understanding domain details
	

________________


Migration Strategy: Iterative Rollout
The platform will be rolled out incrementally to minimize risk and validate value at each stage. Based on existing planning and team readiness, we prioritize vertical blending first (carousel-level optimization has clearer business impact), then horizontal blending in two phases (organic items first, then ads integration).


________________


Phase 1: Vertical Blending (Carousel-Level Optimization)
Timeline: Weeks 1-12 (Q1-Q2 2026)


Objective: Unify carousel-level ranking with value function framework, replacing fragmented heuristic-based vertical positioning with principled optimization.


Key Deliverables:


1. Unified Value Function for Carousels


   * Implement pImp × pAct × vAct framework at carousel level
   * Carousel pAct: aggregate engagement metrics (CTR, CVR) for entire carousel
   * Carousel vAct: expected GOV + strategic value (NV trials, ads revenue per carousel)
   * Calibrate existing carousel scores (UR engagement, VP profit, Merchandising MAB) to comparable scale


2. Constrained Optimization Engine


   * Implement Lagrangian relaxation-based blending algorithm
   * Add diversity constraints (Rx vs NV balance, content type distribution)
   * Add quality constraints (minimum engagement threshold per carousel)
   * Replace existing vertical boosting/pinning with principled λ-based adjustments


3. Self-Service Experimentation Platform


   * Declarative configuration DSL for blending parameters
   * Automatic traffic splitting and experiment bucketing
   * Stage-wise logging for carousel-level decisions
   * Experiment dashboard with carousel-level metrics


4. Migration from Legacy Systems


   * Migrate existing vertical blending logic (VerticalBlending.kt) to new framework
   * Replace hardcoded vertical boost weights with configurable λ parameters
   * Eliminate manual DV setup for carousel positioning experiments
   * Validate neutral or positive impact on key metrics (CTR, CVR, GOV)


Success Criteria:


* 2+ partner teams (NV, Merchandising) launch carousel positioning experiments independently
* Experiment launch time reduced from 2-3 weeks to 3-5 days for carousel-level changes
* No statistically significant regression in homepage engagement metrics


Risks & Mitigations:


* Risk: Calibration drift across carousel types → Mitigation: Continuous calibration monitoring + weekly recalibration
* Risk: Partner team resistance to centralized blending → Mitigation: Preserve partner control via λ parameter tuning APIs


________________


Phase 2: Horizontal Blending - Organic Items (Store/Item-Level Optimization)
Timeline: Weeks 13-24 (Q2-Q3 2026)


Objective: Implement store/item-level calibration and blending within carousels, optimizing for business goals (e.g., Variational Profit or GOV + LTV) with principled constraints.


Key Deliverables:


1. Item-Level Value Function Framework


   * Extend value function to store/item level: pImp_item × pAct_item × vAct_item
   * Item pAct: pCTR, pCVR from organic rankers (UR, MoE, etc.)
   * Item vAct: Variational Profit (VP) for stores, or configurable objective function
      * VP = f(GOV, dasher_cost, commission, strategic_value)
      * Enable partner teams to contribute custom vAct models via plugin API


2. Calibration Infrastructure for Items


   * Isotonic regression to calibrate pCTR/pCVR across different rankers
   * Validate calibration quality: predicted probabilities match observed rates
   * Cross-carousel calibration: ensure stores ranked fairly across Rx vs NV carousels
   * Calibration drift detection and automatic recalibration


3. Horizontal Blending with Constraints


   * Implement greedy blending within each carousel based on expected value
   * Add diversity constraints:
      * Category diversity (e.g., max 3 stores from same cuisine)
      * Brand diversity (e.g., max 2 stores from same chain)
      * Price range diversity
   * Add quality constraints:
      * Minimum store rating threshold
      * Minimum expected engagement (pCTR × pCVR ≥ threshold)
   * Add business constraints:
      * Strategic boosting for new stores (cold-start)
      * Campaign-specific promotion slots with opportunity cost tracking


4. Opportunity Cost & Counterfactual Analysis


   * Log counterfactual rankings: what would have ranked without boosting/constraints?
   * Compute opportunity cost: value lost due to manual interventions
   * Provide transparency to partner teams: "Boosting store X displaced store Y, costing $Z in expected value"


Success Criteria:


* VP optimization shows statistically significant improvement in GOV or configured business metric
* 30%+ of hardcoded store-level heuristics eliminated (inflation multiplier, program boosting)
* Partner teams can launch store-level blending experiments independently via config changes


Example Use Case: Optimize for Variational Profit


For each store i in carousel:
  VP_i = pCVR_i × (expected_GOV_i - expected_dasher_cost_i)
         × commission_multiplier + strategic_value_i


Rank stores by: pImp_position × pCTR_i × VP_i


Subject to:
  - Diversity: max 3 stores per cuisine category
  - Quality: pCVR_i ≥ 2.5%
  - Boosting: new stores get λ_cold_start boost

Risks & Mitigations:


* Risk: VP optimization degrades user engagement → Mitigation: Add CVR constraint to maintain quality floor
* Risk: Complex vAct models slow down serving → Mitigation: Pre-compute vAct offline, cache in real-time serving


________________


Phase 3: Horizontal Blending - Ads Integration (Item-Level with Ads)
Timeline: Weeks 25-36 (Q3-Q4 2026)


Objective: Incorporate Ads into horizontal blending with bidding, pacing, and ad load management, enabling fair competition between organic stores and ads at the item level within carousels.


Key Deliverables:


1. Unified Ads + Organic Item Blending


   * Extend horizontal blending to include both organic stores and ads in same carousel
   * Ads value function: pImp_item × pCTR_ads × (bid_amount + opportunity_cost)
   * Fair comparison: calibrate ads pCTR to organic pCTR scale
   * Greedy merge: interleave ads and organic stores by expected value


2. Ad Load Management & Constraints


   * Dynamic ad load cap based on carousel quality:
      * High-engagement carousels: allow up to 30% ads
      * Low-engagement carousels: cap at 10% ads to protect UX
   * Position-dependent ad load: fewer ads in top positions
   * Carousel-level ad budget: limit ads per carousel to prevent spam


3. Bidding & Pacing Integration


   * Ads team provides bid amount + pacing state via API
   * Platform applies pacing adjustment: reduce ad value if budget nearly exhausted
   * Real-time feedback: notify Ads team of ad delivery rate for pacing updates
   * Second-price auction logic: ad pays (next_highest_bid + $0.01) if wins slot


4. Opportunity Cost for Ads Insertion


   * Compute counterfactual: what organic store was displaced by ad?
   * Log opportunity cost: value_of_displaced_organic_store
   * Report to Ads team: ads are only winning when bid > displaced organic value
   * Transparency for business stakeholders: understand ads/organic trade-off


Success Criteria:


* Ads compete fairly with organic: high-value ads rank high, low-value ads filtered
* Ad load stays within configured caps with no manual intervention
* Ads revenue increases while maintaining or improving organic engagement (CTR, CVR)
* Partner teams (Ads + HP) can independently tune ad load and bidding strategies


Example Flow:


Carousel: Organic Rx Stores + Ads


Organic Stores (from UR):
  Store A: pCTR=5%, VP=$2.50 → Expected Value = 0.05 × $2.50 = $0.125
  Store B: pCTR=4%, VP=$3.00 → Expected Value = 0.04 × $3.00 = $0.120
  Store C: pCTR=3%, VP=$2.00 → Expected Value = 0.03 × $2.00 = $0.060


Ads (from Ads Ranker):
  Ad 1: pCTR=2%, Bid=$5.00 → Expected Value = 0.02 × $5.00 = $0.100
  Ad 2: pCTR=1.5%, Bid=$4.00 → Expected Value = 0.015 × $4.00 = $0.060


Greedy Blending (by Expected Value):
  Position 1: Store A ($0.125)
  Position 2: Store B ($0.120)
  Position 3: Ad 1 ($0.100)       ← Ad wins position 3
  Position 4: Store C ($0.060)
  Position 5: Ad 2 ($0.060)       ← Tied, apply diversity constraint (max 30% ads)


Opportunity Cost Logging:
  Ad 1 displaced: Store C (would have been position 3)
  Opportunity cost: $0.060 organic value → Ads paid $0.100, net gain = $0.040

Risks & Mitigations:


* Risk: Ads dominate carousel due to high bids → Mitigation: Ad load caps + quality constraints (min pCTR for ads)
* Risk: Ads team loses control over ad placement → Mitigation: Provide pacing APIs and real-time feedback dashboards
* Risk: Calibration drift between ads pCTR and organic pCTR → Mitigation: Continuous cross-calibration validation


________________


Phase 4: Advanced Features (V2+)
Timeline: Weeks 37-52+ (Q4 2026 and beyond)


Objective: Add context-aware personalization, cold-start optimization, and advanced value functions to improve ranking quality and business impact.


Key Deliverables:


1. Context-Aware Personalization


   * Segment value functions by:
      * Geographic area (district, city, region)
      * Consumer cohort (new vs. existing, high vs. low frequency)
      * Session context (time of day, day of week, weather)
   * Learn personalized λ parameters for each segment
   * A/B test personalized vs. universal blending


2. Cold-Start with MAB & Pacing


   * Multi-Armed Bandit (MAB) for new content with no historical data
   * Thompson Sampling or UCB for exploration-exploitation trade-off
   * Pacing controls: gradually ramp up exposure for new content
   * Exposure guarantees: ensure minimum impressions for cold-start items


3. Advanced Value Functions


   * Long-term value (LTV): incorporate user lifetime value into vAct
   * Strategic value: trials, new vertical adoption, brand awareness
   * Cross-surface value: optimize for HP + VLP + Search jointly
   * Session-level value: maximize value across entire user session, not just single page


4. Reinforcement Learning for Dynamic Optimization


   * Replace static λ parameters with RL-learned policy
   * Contextual bandits for real-time λ adaptation
   * Multi-objective RL: balance short-term revenue + long-term engagement
   * Online learning: continuously update policy based on live feedback


Success Criteria:


* Personalized blending shows lift in engagement and revenue for target segments
* Cold-start content reaches desired exposure without manual pinning
* RL-based optimization outperforms static λ tuning in offline simulation


________________


Phasing Summary
Phase
	Focus
	Timeline
	Key Outcome
	Phase 1
	Vertical Blending (Carousel-Level)
	Weeks 1-12
	Partner teams launch carousel experiments independently; DV setup time reduced to 3-5 days
	Phase 2
	Horizontal Blending - Organic Items
	Weeks 13-24
	VP optimization live; 30%+ heuristics eliminated; item-level calibration validated
	Phase 3
	Horizontal Blending - Ads Integration
	Weeks 25-36
	Ads compete fairly with organic; ad load managed systematically; revenue lift without UX degradation
	Phase 4
	Advanced Features (Personalization, Cold-Start, RL)
	Weeks 37-52+
	Context-aware ranking; MAB cold-start; RL-based optimization
	

Each phase is independently shippable and provides measurable value, reducing execution risk while building platform capabilities progressively.


________________


Technical Design
Value Function Framework
[To be written]
Two-Layer Blending Architecture
[To be written]
Calibration & Normalization System
[To be written]
Experimentation & Configuration Management
[To be written]
Cold-Start, Constraints, and Advanced Features
[To be written]


________________


System Architecture
[To be written]


________________


Ownership & Collaboration Model
Overview
The Unified Blending Platform operates on a platform-first ownership model where the HP team owns the infrastructure, APIs, and core optimization logic, while partner teams (NV, Ads, Merchandising Platform) own domain-specific candidate generation, ranking models, and business objectives.


This model balances centralized control (for consistency, quality, and velocity) with distributed contribution (for domain expertise and innovation).


Key Principles:


1. HP is the DRI (Directly Responsible Individual) for platform health, API stability, and cross-team coordination
2. Partner teams are co-owners of the value function, constraints, and experimentation strategy
3. All teams can contribute to platform infrastructure through design collaboration and gated PR reviews
4. Decisions are transparent and driven by data, with escalation paths for conflicts


________________


Responsibility Matrix
Component
	HP Team
	Partner Teams (NV, Ads, Merch)
	Collaboration Model
	Platform Infrastructure
	✅ Owns
	🤝 Contributes
	Partner teams propose features; HP gates design + PR review
	Blending Engine (Vertical + Horizontal)
	✅ Owns
	📖 Consumes
	HP implements; partners provide feedback on API design
	Value Function Framework
	✅ Owns
	🤝 Co-designs
	HP implements framework; partners define domain-specific value models (VP, eCPM, engagement)
	Calibration Infrastructure
	✅ Owns
	🤝 Co-designs
	HP builds calibration tooling; partners validate calibration accuracy for their content types
	Experimentation Platform
	✅ Owns
	📖 Consumes
	HP provides DSL + APIs; partners configure experiments independently
	Candidate Generation
	📖 Consumes (for Rx organic)
	✅ Owns
	Each team owns their candidate providers (UR for Rx, Ads Ranker, NV Ranker, Merch suggestions)
	Ranking Models (pCTR, pCVR, vAct)
	✅ Owns (for Rx organic)
	✅ Owns (for their domains)
	Each team trains and deploys their own prediction models
	Business Objectives & Constraints
	✅ Owns (for Rx organic)
	✅ Owns (for their domains)
	Partner teams define objectives (VP, eCPM, growth targets); HP enforces via constraints
	Platform Health Monitoring
	✅ Owns
	📖 Views
	HP monitors latency, error rates, experiment coverage, platform adoption
	Domain-Specific Metrics
	✅ Owns (Rx CTR, CVR, GMV)
	✅ Owns (Ads CTR, fill rate, NV GMV)
	Each team owns and monitors their business metrics; shared in dashboards
	Documentation & Onboarding
	✅ Owns
	🤝 Contributes
	HP maintains platform docs; partners document their integration patterns
	

Legend:


* ✅ Owns: Decision-maker and executor; responsible for quality, timelines, and outcomes
* 🤝 Co-designs or Contributes: Active participant in design and implementation; proposes changes but does not unilaterally decide
* 📖 Consumes or Views: User of the component; provides feedback but does not build or maintain


________________


Contribution & Decision-Making Process
1. Feature Proposals
When a partner team wants to add a new feature (e.g., Ads team needs pacing constraints, NV team wants geographic segmentation):


2. Propose: Partner team creates a design doc outlining:


   * Problem statement and use case
   * Proposed API changes or new components
   * Impact on platform (latency, complexity, experiment surface area)
   * Alternative approaches considered


2. Collaborate: HP team reviews and schedules design review meeting (sync or async)


   * Joint discussion of trade-offs, edge cases, and implementation strategy
   * HP validates feasibility, cost, and alignment with platform roadmap


3. Decide: HP team makes final call on whether to prioritize, with input from partner teams


   * If approved: work is added to roadmap with clear owner (HP or partner team contributor)
   * If deferred: documented rationale and revisit timeline


4. Implement: Code contribution (by HP or partner team)


   * All PRs reviewed and approved by HP team before merge
   * HP ensures code quality, testing, and documentation standards


5. Launch: Feature rolled out via experiment or gradual ramp-up


   * HP team monitors platform health; partner team validates feature behavior
2. Decision Authority
* Platform Direction: HP team owns final decision, but actively solicits partner feedback
* Domain-Specific Logic: Partner teams own decisions (e.g., Ads bidding strategy, NV ranking model updates)
* Shared Components (e.g., value function weights, calibration strategy): Joint decision through design review and data-driven analysis
* Conflicts: Escalate to leadership (HP Director + Partner Director) for alignment
3. Code Contribution Guidelines
* All teams can contribute to platform infrastructure (blending engine, calibration, experimentation tooling)
* HP gates all changes through:
   * Design review (for new features or architectural changes)
   * PR review (for code quality, tests, performance, documentation)
   * Integration testing and experiment validation
* Fast-track path: For urgent bug fixes or small improvements, HP provides expedited review (24-48 hour SLA)


________________


Cross-Org Staffing Model
To ensure deep collaboration and shared context, partner teams dedicate embedded contributors to work closely with HP on platform development.
Current Staffing (Starting with NV Team)
* NV Team Contribution:


   * 1 MLE (Machine Learning Engineer): Co-design value function, calibration, and experimentation framework
   * 1 BE (Backend Engineer): Contribute to blending engine, API integration, and performance optimization


* HP Team:


   * Core platform team (3-5 engineers): Owns infrastructure, blending engine, calibration, experimentation platform
   * Organic Rx ranking team: Acts as a "partner team" for Rx content, using the platform as a consumer
Expected Staffing from Other Partner Teams
* Ads Team: 1 MLE + 1 BE (for ads integration in Phase 3)
* Merchandising Platform Team: 1 MLE (for merch-specific value functions and constraints)
Collaboration Model
* Weekly Sync: HP + embedded contributors review roadmap, resolve blockers, align on priorities
* Design Reviews: Joint sessions for major features (value function updates, new constraint types, API changes)
* On-Demand Support: HP provides integration support, debugging help, and experiment guidance via Slack + ticketing system
* Shared Documentation: Confluence space for platform specs, API references, runbooks, and onboarding guides


________________


Metrics & Monitoring Ownership
Clear separation of concerns ensures accountability without overlap:
HP Team Owns: Platform Health Metrics
* Latency: p50, p95, p99 for blending engine, calibration, and candidate retrieval
* Error Rates: 5xx errors, timeouts, invalid configurations
* Experiment Coverage: % of traffic in experiments, experiment launch velocity (# experiments per quarter)
* Platform Adoption: # of partner teams integrated, # of experiments launched via platform
* Code Quality: Test coverage, code complexity, heuristic reduction


Monitoring Tools: Datadog, Prometheus, Grafana dashboards Alerting: HP on-call rotation for platform incidents
Partner Teams Own: Domain-Specific Business Metrics
* Ads Team: Ads CTR, fill rate, eCPM, revenue, pacing stability
* NV Team: NV GMV, order conversion, store impressions, engagement depth
* Merchandising Platform: Merch CTR, margin-weighted CVR, product adoption
* Rx Organic (HP): Rx CTR, CVR, GMV, session retention


Monitoring Tools: Each team's existing dashboards + shared Looker/Tableau reports Alerting: Partner teams own on-call for domain-specific metric regressions
Shared Metrics: Whole-Page Optimization
* Homepage-Level Metrics: Total GMV, revenue per session, long-term retention, marketplace health
* Joint Review: Weekly metric review meetings with HP + all partner teams
* Attribution: Counterfactual analysis to attribute lift to blending changes vs. domain-specific improvements


________________


Change Management: Breaking Changes
To ensure stability and predictability, breaking API changes follow a structured process:
1. Definition of Breaking Change
A change is considered breaking if it:


* Changes API contracts (request/response schemas, endpoint URLs)
* Removes or renames configuration parameters
* Alters semantic behavior in a non-backward-compatible way
* Requires partner teams to update their integration code
2. Process for Breaking Changes
3. Announce: HP team announces breaking change one quarter (3 months) in advance


   * Document: What's changing, why, and migration path
   * Notify: Email + Slack announcement + doc shared in design review


2. Migration Support: HP team provides:


   * Migration guide with code examples
   * Backward-compatible shim layer (if feasible) for 1-2 quarters
   * Office hours for integration support


3. Validation: Partner teams test migration in staging environment


   * HP provides sandbox with breaking change enabled
   * Partner teams validate no regressions in their experiments


4. Rollout: Gradual rollout with kill switch


   * Week 1-2: Internal HP team (dogfooding)
   * Week 3-4: 10% traffic
   * Week 5-8: 50% traffic
   * Week 9-12: 100% traffic (old API deprecated)


5. Deprecation: Old API turned off after migration period (1-2 quarters post-announcement)
6. Exceptions
* Critical Bug Fixes: May require immediate breaking change; HP notifies partners ASAP and provides expedited migration support
* Non-Breaking Enhancements: New optional parameters, additional APIs, or new features do not require advance notice


________________


Governance & Conflict Resolution
1. Experiment Prioritization
* Default: Platform has sufficient traffic to run concurrent experiments by different teams


   * Experiments are orthogonal if they target different carousels, content types, or user segments
   * Example: NV can test carousel ordering while Ads tests ad load simultaneously


* Conflict Scenario: When experiments overlap and are mutually exclusive (e.g., both want to test different value function weights):


   1. Negotiate: Teams discuss priority, business impact, and experiment timeline
   2. Sequence: Run experiments sequentially if both are high-priority
   3. Escalate: If no agreement, escalate to leadership (HP Director + Partner Director) for prioritization decision
4. Platform Roadmap
* Quarterly Planning: HP team drafts roadmap based on:


   * Platform goals (adoption, velocity, code quality)
   * Partner team feature requests
   * Technical debt and performance improvements


* Review & Alignment: Roadmap reviewed with partner teams in quarterly planning meetings


   * Partner teams vote on priority (high/medium/low) for proposed features
   * HP team weighs votes + platform needs and finalizes roadmap


* Mid-Quarter Adjustments: Urgent requests (P0 bugs, high-impact features) can be added mid-quarter with HP approval
3. Success Criteria Ownership
From the Vision & Principles section, success criteria are owned as follows:


Metric
	Owner
	Accountability
	Experiment Launch Velocity (3-5 days)
	HP Team
	Ensure platform tooling enables fast DV setup and experiment launch
	Platform Adoption (2+ teams, 2-3 experiments per quarter)
	HP Team
	Drive partner team onboarding and integration
	Code Quality (30%+ heuristic reduction, 20%+ code reduction)
	HP Team + Partner Teams
	HP measures platform code quality; partner teams measure domain code quality
	Experimentation Throughput (3-5x increase)
	HP Team
	Enable more experiments via platform capabilities
	Business Impact (GMV lift, revenue increase, engagement improvement)
	Partner Teams
	Each team tracks domain-specific impact; shared in joint metric reviews
	

Reporting Cadence:


* Weekly: Platform health metrics (latency, errors, experiment coverage)
* Monthly: Adoption and velocity metrics (# experiments launched, DV setup time)
* Quarterly: Business impact review (GMV, revenue, engagement) across all partner teams


________________


Summary
The Ownership & Collaboration Model ensures:


1. Clear accountability: HP owns platform; partner teams own domain logic
2. Shared contribution: All teams can propose and build features through gated collaboration
3. Efficient decision-making: HP has final say on platform direction; joint decisions on shared components
4. Predictable change management: Breaking changes announced 1 quarter in advance with migration support
5. Transparent metrics: Platform health owned by HP; business impact owned by partner teams; whole-page metrics reviewed jointly


This model enables fast iteration (partner teams launch experiments independently) while maintaining platform stability (HP gates infrastructure changes) and business alignment (shared metrics and quarterly planning).


________________


Phased Implementation Strategy
[To be written]


________________


Engineering Efficiency Impact
[To be written]


________________


Appendix
Industry References
[To be written]
Technical Deep-Dives
[To be written]


________________




Document Version: 1.0 Generated with: Claude Code