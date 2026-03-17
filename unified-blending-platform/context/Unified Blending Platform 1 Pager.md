One Pager
One Pager: Unified Blending Platform (UBP)
Yu Zhang Feb 26, 2026
Executive Summary
The Ask
Approve Unified Blending Platform: systematic, ML-driven framework replacing fragmented Homepage blending with principled whole-page optimization.


Timeline: 6  months (5 phases)[a][b] | Resources: 3 HP MLE/BE 100% bandwidth + 3-4 partner MLE/BE from NV/Ads/Merch) with 50% bandwidth (Rough Estimation pending detailed scope and LOE sizing)
Why Now: The Cost of Fragmentation
Homepage evolved Rx → multi-business (Rx + NV + Ads + Merchandising) faster than infrastructure adapted, creating unsustainable fragmentation:


* Engineering: 2-3 weeks[c][d][e][f] to setup up new experiments (should be 2-3 days); 30+ unmaintained heuristics; 15+ files modified per experiment
   * Pain Points
   * How UBP Boost the velocity
* Organizational: Merch Platform/Ads rebuilding blending work HP already built; partners blocked on HP
* Competitive: Pinterest/Meta/Google achieving meaningful efficiency gains; we iterate at 1/10th velocity
The Solution
Unified Blending Platform:


1. Auto-calibration: all input scores with a quantitative physical measure need to be calibrated through both one-off offline as well as continuous online.
2. Offline Simulation: quick understanding and efficient selection of candidate model and parameters 
3. Platformized Experimentation: Modular Ranking Interface with plug-and-play, Config-Driven Experimentation, Built-In Observability, Safe Parallel Isolation. 
4. Unified Value Function[g][h][i]: All content (Organic/Ads/Merch) evaluated on comparable scale (Expected Value = pImp × pAct × vAct)
5. Two-Layer Optimization: Simultaneous carousel-level[j][k] + item-level whole page optimization; Ads compete fairly vs. inserted post-ranking
6. Clear Ownership & Separation of Concerns: HP owns platform; partners own domain models
Impact
Engineering Efficiency: 2-3 weeks → 2-3 days experiment launch; eliminate 30+ heuristics; 2+ partner teams self-serve


Business Value: Whole-page optimization; principled revenue/engagement trade-offs; personalization at scale; transparent opportunity cost


UX Protection: Maintain or improve CTR/CVR/retention—non-negotiable guardrail
What We Need
1. Approve: 6 month initiative, phased rollout
2. Resources: 3 HP engineers + NV/Ads/Merch embedded contributors
3. Alignment: Adopt and collaborate on HP UBP platform without other duplicated efforts
4. Ownership: HP as Platform DRI; joint ownership for value function
Problem Statement
The Core Problem
Multiple ranking systems (Organic UR, Ads ranker, VP optimizer, Merchandising MAB) operate independently with incomparable scores, making principled blending impossible. Ads inserted after organic ranking—highest-value Ads might not be presented at the top of HP.


Result: Optimizing page pieces, not whole pages.
Four Critical Failures
1. Configuration Hell: 2-3 weeks to launch experiment (e.g. NV SR, Blending v0); 15+ files modified; manual DV with nested references; HP overwhelmed with plumbing vs. ML innovation.


2. Technical Debt: 30+ hardcoded heuristics (inflation multiplier, NV boosting, calibration multipliers) with no measurement; no systematic value function; no stage-wise logging; cannot compute counterfactuals.


3. Organizational Inefficiency: without a centralized CBP, partner teams such as Merch Platform[l][m] need to invest in duplicated/suboptimal short-term solutions ;HP’s bandwidth and coordination bottleneck for all partners; slow decision-making with ad-hoc Ax review needed.


4. Missing Capabilities: Cannot personalize by cohort/context with fast online parameter tuning; cannot measure opportunity cost for boosting/pinning when really needed; cannot systematically cold-start (manual pinning for weeks without graduation policy); no observability into E2E pipeline at different stages.
The Trajectory
* Engineering Cost: 20-30% HP time on plumbing/firefighting vs. ML innovation
* Innovation Cost: Experimentation stalled; partners blocked
* Competitive Risk: Industry leaders 15-30% efficiency gains; we're 1/10th velocity
* Organizational Waste: Teams duplicating infrastructure


Opportunity: Unify platform → eliminate heuristics, reduce coordination, unlock efficiency, enable self-service, empower collaboration, establish scalable ML innovation.
Solution Overview
What Is It?
Systematic, ML-driven framework replacing fragmented ranking with whole-page optimization. HP owns infrastructure (value function, calibration, experimentation); partners own domain logic (models, objectives).


Key Shift: "Everyone does everything" → Platform-first with clear boundaries.
Core Architecture
Unified Value Function:


Expected Value = pImp × pAct × vAct
  pImp = Probability of Impression (position-dependent)
  pAct = Probability of Action (calibrated across content types)
  vAct = Value of Action (GOV, FIV, Ads revenue, NV trial value, other strategic value)

Two-Layer Blending:


* Horizontal (Item/Store-Level): Ranks item/stores within carousels by expected value + diversity constraints, for 1) scalable solution to optimize for business goals (e.g. VP), 2) principled mixed blending organic and Ads (Ads compete fairly with Organic, not inserted after.)
* Vertical (Carousel-Level): Ranks carousels across positions by expected value + quality/business constraints + other business logic/needs
How It Solves Problems
Problem
	Solution
	Result
	Configuration Hell
	Declarative config; self-service platform; automatic traffic splitting
	2-3 weeks → 1-2 days
	Technical Debt
	Unified value function; 
automatic calibration;
stage-wise logging; counterfactual analysis
	Eliminate 30+ heuristics
	Organizational Inefficiency
	Clear ownership; shared platform; plugin architecture
	Stop multiple month duplication work
	Missing Capabilities
	Cx and Mx cold-start; opportunity cost tracking; online tuning with context personalization; E2E observability
	Enable personalization at scale; measure trade-offs
	

Ownership: HP (blending engine, calibration, experimentation, monitoring) | Partners (candidate generation, ranking models, domain specific value models, domain metrics)


Technical Approach
High-level Overview
Calibration is Critical Path: Hardest challenge = ensuring pAct predictions from different models calibrated to comparable scales. HP builds centralized calibration infrastructure (e.g. isotonic regression, continuous validation). Get this wrong → entire platform suboptimal.


Constrained Optimization is Tractable: Use Lagrangian relaxation (Pinterest/Google/Uber approaches) converting complex constrained problems into greedy blending with learned λ parameters and keep iteration from original heuristic to RL


Risk Mitigation Through Phasing: Phase 1 (vertical blending, carousel-level, lower risk) → Phase 2 (organic horizontal blending for VP, lower risk) → Phase 3 (systematic cold-start, lower risk)  → Phase 4 (merch platform integration, median risk) → Phase 5 (ads integration, high risk). Each phase is independently shippable with measurable value.
Key Technical Investments
Infrastructure: Calibration service, config management, enhanced logging
ML/Optimization: Value function framework, constrained optimization algorithms, offline simulation
Observability: Experiment dashboards, opportunity cost tracking, drift detection
Business Impact & Success Metrics
Engineering Efficiency
* Experiment Velocity: 2-3 weeks → 1-2 days (10x+ throughput, unless the total available experiment traffic becomes the only bottleneck)
* Technical Debt: Reduce complexity and improve quality and reliability by eliminating 30+ heuristics, (HP shifts 30% firefighting → Platform improvement and ML innovation)
* Platform Adoption: 2+ partner teams self-serve
Business Value
* Whole-Page Optimization: Fair Ads/Organic competition; principled trade-offs (industry benchmark: X% efficiency gains)
* Personalization: Context-aware tuning for value functions by geo/cohort/intent; systematic cold-start (improved engagement/retention); velocity boost for testing SoTA models and new objectives in different domains. 
* Transparency: E2E score/decision tracking at all stages; opportunity cost tracking; counterfactual analysis; data-driven decisions
Cx Experience Protection
Guardrails: Maintain or improve HP relevance/App OR/Retention; fail-safe fallback
Success Criteria: Efficiency and Business gains while protecting or improving Cx experience.
Scope & Resources
Scope
In: Homepage blending (vertical + horizontal) for Rx/NV/Ads/Merch; unified value function; auto calibration; self-service experimentation; observability; automated decision-making framework


Out: Domain/Surface specific functionality; Cross-surface optimization; RL (Future); Real-time model retraining
Note, this doesn’t improve the velocity of the individual offline model development, but hugely improves the velocity afterwards and scalability to quickly test more models and ideas.
Resource Ask
Team: HP Core (2 MLE + 1 BE + 1 PM) for 6 months | Partner Contributors: NV (1 MLE + 1 BE, Phases 1-2), Ads (1 MLE + 1 BE, Phase 3), Merch (1 MLE/BE, Phases 1-2) each for 3 months (Rough Estimation pending detailed scope and LOE sizing)
Leadership Support
Approval: Endorse the proposed resource and timeline and phased rollout


Alignment: Partner leadership commits embedded contributors; stop duplicated development


Authority: HP as DRI for platform; joint ownership for value function; clear escalation path


Accountability: HP owns experiment velocity + platform adoption | Partners own domain metrics | Shared own unified value function and Cx experience protection






How UBP Boost the Velocity
How UBP Boost the Velocity
Plug-and-Play Predictors
* Predictors registered via standard interface
* No BE changes for new variants
* Runtime configuration for parameter tuning
* Built-in feature validation and logging
Impact:
* Adding predictors drops from 2 weeks → 1–2 days
* Eliminates repeated plumbing work
Config-Driven DV & Traffic Management
* Self-serve traffic splitting
* Dynamic start/stop/reshuffle
* Isolation between parallel experiments
* No manual DV surgery
Impact:
* Eliminates E6-level review cycles
* Enables safe parallel experimentation
* Removes 1+ week coordination overhead
Automatic Logging & Observability
* Standard schema for predictions, features, assignments
* Real-time dashboards
* No manual instrumentation
Impact:
* Removes 1 week of logging work per experiment
* Faster debugging and iteration
* Lower risk of blind spots
Quantified Impact
Activity
	Today
	With UBP
	Add simple predictor
	~2 weeks
	1–2 days
	DV setup
	1–2 weeks
	0 (config)
	Logging instrumentation
	~1 week
	Automatic
	Traffic split
	Senior MLE-heavy
	Self-serve
	Cross-team alignment
	~1 week
	Minimal
	

If ~70–80% of effort is infra overhead for setup experiment, removing it transforms: 2–3 weeks → 2–3 days. Please note, this doesn’t improve the velocity of the individual offline model development, but hugely improves the velocity afterwards and scalability to quickly test more models and ideas.




Pain Points
1️⃣ Experiment Setup Is Engineering-Heavy (Not Just Modeling)
Even “simple” experiments require:
* BE changes
* DV wiring and setup
* Manual wiring of predictors and Logging instrumentation
* Cross-system coordination
This turns what should be a parameter tweak into 2+ weeks of engineering work.
The bottleneck is not modeling — it’s infra + coupling.
2️⃣ Deeply Coupled & Nested Logic Across Systems
Example: NV SR
* DV setup issue blocked experiment for 2 weeks
* 2 MLE + 1 BE involved
* Complex nested logic between HP and VLP ranking
* Debugging tightly coupled code paths
Root issue: ranking logic and experimentation logic are intertwined and not modular.
3️⃣ Predictors Are Not Plug-and-Play
Examples:
* VP optimization: 2 weeks to add simple linear parameter combination
* MAB predictor: 2 weeks infra + 1 week logging
* Real-time feature and entity bugs across multiple rounds
Common themes:
* Adding predictors requires code changes
* Feature plumbing is manual
* Logging is not standardized
* Each experiment rebuilds the same scaffolding
4️⃣ DV Setup Is Complex and Manual
Split traffic for UR:
* Extremely complex DV configuration
* Needed two E6 MLEs Multiple review rounds
* Hard to dynamically start/stop/reshuffle experiments
Root issue: experimentation framework lacks:
* Self-serve configuration
* Dynamic traffic routing
* Isolation between experiments
* Clear abstraction boundaries
5️⃣ Coordination & Alignment Cost
* Multiple experiments in parallel add 1 extra week
* Cross-team dependencies
* Ax manual reviews
* Risk of breaking coupled logic
This is a hidden tax not captured in ENG cost.


Related Docs
[Draft] Unified Blending Platform
[a]One question we will get given this duration is whether to do this in the existing system, in Pedregal, or both.  It will be good to include a section w/ your pov on how you propose to balance building for now vs. future
[b]Yes. The timeline is not accurate or finalized here. With that said, I do expect we need to divide phases into pre-PD and post-PD, and I have invited @yu.zhu@doordash.com who is the DRI on PD side to help review as well.
[c]What is the biggest contributor to “2-3 weeks”? Is it the time for parameter tuning to identify experiment candidates? Or is it the time for cross org discussion?
[d]+1, why is it so slow today and why we believe the new framework can make it 5x faster
[e]No. I am only talking about ENG cost here. We have not count the additional alignment and coordination cost across teams when multiple experiments need to run at the same time, which normally is another week. Let alone the online experiment testing time.


If we are just testing a UR variant without any BE or DV logic or change, that's simple, but whenever we go beyond that, the cost is significantly higher. Recall there is a time in 2025 that multiple projects were delayed due to this.


Here are some concrete examples for why it would cost 2+ weeks:
1) NV SR: the DV setup blocks the experiment by 2 weeks from cost 2 MLE and 1 BE to debug and fix the nested and coupled code logic between HP and VLP ranking
2) VP optimization: 2 weeks MLE time to setup and test additional predictors, adding heuristic logic and setup DV to start experiment for a simple linear parameter combination, no logs available now and is still WIP
3) Adding MAB predictor: 2 weeks time to setup additional predictor, tested multiple rounds to fix entity and real-time feature for the model, 1 week to adding additional logs
4) Split traffic for UR: super complex and nested DV setup when we want to split the traffic to run multiple UR experiments at the same time, with the freedom to start, end and reshuffle anytime. It took two E6 MLE to live review multiple rounds and tests.


With the proposed UBP, MLEs is expected to make similar changes with just config/runtime changes without even need to touch the BE or manually setup DV and figure out the complex underlined logic.
[f]Added two Tabs with links in case audience are interested in more details.
[g]Who decides the “vAct”? How to make sure “vAct” from all teams are fair?
[h]In the near future, we will have videos. TS will be more appropriate for video. Who decides how to incorporate TS in VM?
[i]How to define or compute vACT is out of the scope of the UBP, it is either defined, online tuned, or directly take from FIV by each partner teams (HP ranking team is also the partner team from this perspective as HP team owns both platform as well as organic ranking at the same time) and need to align with all stakeholders.
UBP takes these as inputs.
[j]Maybe a even bigger question: should we also consider blending b/w carousels and stores?
[k]Maybe we can use a "vertical slot" instead of carousel here as we also have standalone campaign, together with spotlight carousel, banner carousel, video carousel, etc. We don't currently have a single store card taking a vertical slot except the store-feed, which by itself is a carousel plotted vertically. If we have any UI change in the future, it will fit into UBP as long as the new content still provide the required meta data.
[l]Will Merch Plat have the same magnitude of stores to horizontal rank as other carousels?


Will Merch Plat always have the appropriate content format to be vertically blended with other carousels?
[m]currently the magnitude is only at few hundreds and it could increase rapidly; however I don't expect that to ge to the same scale with the stores.


For those that have appropriate content/UI format, we can use the vertical blending layer to address them; for those new content/UI, we can have UBP's cold-start component to address them.