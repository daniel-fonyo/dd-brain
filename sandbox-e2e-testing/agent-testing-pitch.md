# Autonomous Agent-Driven Testing for Feed-Service Homepage

## The Problem

Homepage development in feed-service is bottlenecked by manual testing. Every change, whether it's ranking adjustments, experiment rollouts, new carousel types, or blending logic, requires an engineer to spin up a sandbox, load the homepage, eyeball the feed, cross-reference pod logs, check Snowflake events, verify experiment enrollment, and repeat for different scenarios. A single change can take 30-60 minutes of manual validation, and it's inconsistent. How thorough the validation is depends on who's doing it and how much time they have.

The homepage is a black box. Ranking inputs go in, a feed comes out, and verifying that the feed is *correct* requires correlating signals across multiple systems (service logs, browser rendering, Snowflake analytics events, experiment platforms) that were never designed to be observed together. This makes testing naturally slow, and the slowness compounds as the volume of homepage asks increases.

**The result:** Testing is the bottleneck. Requests pile up, regressions risk slipping through, and engineers spend a disproportionate amount of time on repetitive validation work instead of building.

<!-- TODO: Add 2-3 specific recent examples. Include turnaround times, regressions caught late, backlog pressure. -->

## Why This Matters Now

The volume of homepage-related asks is increasing: new verticals, ranking experiments, blending changes, cross-vertical expansion. If we try to absorb this with the current manual workflow, either we slow down (testing bottlenecks delivery) or we cut corners (regressions slip through). Neither is acceptable.

**We need to zoom out.** Instead of grinding through each ask with the current workflow, we should improve the workflow itself. Invest in testing infrastructure that unlocks higher throughput and velocity for *all* homepage work going forward.

## The Solution: Autonomous Agent-Driven Sandbox Testing

An AI agent that has the tools to autonomously validate homepage changes end-to-end. The engineer describes *what* to test. The agent handles *how*.

### How It Works

The engineer sits down with the agent for 5-10 minutes and collaboratively builds a testing plan: what the change does, what success looks like, what signals to check. Then the agent takes over.

![Pipeline diagram](../sandbox-e2e-testing/pipeline-diagram.png)
<!-- TODO: Embed the pipeline diagram image here -->

**Context.** The agent starts with a testing plan and the code change (a diff). It understands what changed and what to validate.

**Code Change to Sandbox.** The agent syncs the diff to a sandbox environment via `devbox run`, deploying the code to a live feed-service pod.

**Sandbox Environment.** The agent drives a real Chromium browser (via Playwright) against the sandbox pod. It navigates the homepage, scrolls, clicks through carousels, and enables debug overlays. These are the same interactions an engineer would perform manually.

**Output Signals.** Three independent signal sources are captured in parallel:
- **Application logs**: pod logs streamed via kubectl, parsed for errors, warnings, and debug output
- **Structured extraction data**: carousel inventory, store IDs, ML ranking scores, and latency, all produced by the agent's browser interactions
- **Snowflake events**: Iguazu analytics events queried to confirm the data pipeline fired correctly

**The Autonomous Loop.** This is the core. The agent operates in a reinforcing flywheel: make a decision (what input to provide), observe the system's reaction (what output was produced), analyze against the plan's success criteria, and decide what to do next. This loop runs autonomously until all criteria are evaluated. The agent can reason about what it sees. If scores look wrong, it can re-check. If a carousel is missing, it can investigate why.

**Results.** The agent produces a PR with full end-to-end testing evidence: session video, structured analysis, log snippets, Snowflake queries, and a pass/fail verdict with reasoning. This is not a unit test result. It is evidence that the homepage *actually works as expected* when a real browser hits it.

### The Vision: From Change to Tested PR

The end-state workflow:

1. Engineer (or anyone) comes to homepage and says: *"I want to make change X to have result Y"*
2. Engineer and agent spend 5-10 minutes collaborating on a testing plan
3. Agent takes over: codes the change, deploys to sandbox, runs the full test loop autonomously
4. Agent returns a PR with the code change AND complete end-to-end testing evidence
5. Reviewer gets strong signal: the change works, here's the proof

Unit tests are already covered by agentic coding workflows. This fills the gap for **integration and end-to-end validation**, the part that has historically required an engineer sitting in front of a browser.

**Future extension:** This same pattern naturally extends to **production shadow testing**, another painfully long and tedious validation workflow we use today. The agent can manage shadow traffic routing, compare sandbox vs. production responses, and report differences, using the same autonomous loop.

### This Exists Today

I've used this workflow successfully 3-4 times on real PRs. Each cycle, I improved the agent's skills: browser interaction patterns, log analysis, error recovery, and report formatting. It gets significantly better with each iteration because corrections are persisted into the agent's playbooks and self-healing protocols.

What's needed now is to **productionalize it**: formalize the skills, close the remaining gaps, and make it available for anyone working on homepage changes.

## What's Built vs. What's Needed

| Capability | Status |
|---|---|
| Sandbox lifecycle management (spin up, sync, tear down) | Built |
| Browser automation (navigate, scroll, click, debug overlays) | Built |
| 3-load homepage testing with cross-load consistency analysis | Built |
| ML score extraction (carousel scores, store scores, second-pass) | Built |
| Pod log capture and analysis (ERROR/WARN, request tracing) | Built |
| Snowflake event 1:1 validation against browser observations | Built |
| PR evidence packaging (video, screenshots, logs, queries) | Built |
| Self-healing browser interactions (adapts when UI changes) | Built |
| **Orchestrator (single-command entry point)** | **Needed** |
| **DV/experiment enrollment assertion** | **Needed** |
| **Ranking order assertion (score monotonicity)** | **Needed** |
| **CI integration (GitHub Actions on every PR)** | **Needed** |

The foundation is solid. The remaining work is formalization and extension, not greenfield.

## Expected Impact

| Metric | Today (manual) | With agent testing |
|---|---|---|
| Time to validate a homepage change | 30-60 min | ~5 min (agent runs autonomously) |
| Confidence in validation | Inconsistent, depends on engineer thoroughness | Consistent, same checks every time, multi-load |
| Regression detection | Caught in review or production | Caught before PR merge |
| Evidence trail | Screenshots in Slack, verbal "looks good" | Structured report with video, logs, queries, and scores |
| Scalability | Linear with engineer time | Parallel. Agent tests while engineer works on next change |
| Onboarding | New engineers struggle to test homepage changes | Agent guides testing plan and handles execution |

## What We Need

- **Engineering time** to productionalize the workflow: build the orchestrator, add remaining assertion layers, integrate with CI
- **Team adoption.** Engineers start using the agent for their sandbox testing, contributing improvements
- **Iteration.** The flywheel improves with use. Each run surfaces new signals, self-healing patterns, and assertion types

## Alignment

This aligns with the broader company investment in agentic engineering workflows. We are applying AI tooling to a real engineering bottleneck, not experimenting in a vacuum but solving the specific problem of homepage validation throughput that constrains our delivery velocity.

---

## Appendix: Example Agent Test Output

The following is a real test output from the agent validating the AFM Unpin Exemption change ([PR #62270](https://github.com/doordash/feed-service/pull/62270)). The agent:

1. Collaborated on a testing plan targeting specific decision points in the ranking code
2. Instrumented the code with temporary debug logs at 4 key locations in `Boosting.kt`
3. Deployed to sandbox and ran 3 structured tests (DV off baseline, DV on treatment, regression check)
4. Captured pod logs, screenshots, and carousel position maps for each test
5. Analyzed results against the plan's pass criteria and produced a verdict

### Test Plan (agent-generated, engineer-reviewed)

The agent identified 4 debug log insertion points targeting the DV toggle, eligibility check, AFM detection logic, and final carousel ordering. Each test had explicit pass criteria defined upfront.

### Sample Result: Test 1, Baseline (DV OFF)

**Debug log output** (captured from pod):
```
[AFM_DEBUG] enable_afm_unpin_exemption DV=control
[AFM_DEBUG] isEligibleForUnpin: carouselId=e25be2d7 verticals=[100322,...] nvEligible=true (AFM!)
[AFM_DEBUG] unpin_check: carouselId=e25be2d7 nvEligible=true afmExempt=false willUnpin=true
[AFM_DEBUG] boosted_order=[4e12d866, cxgen:0-8, sponsored_carousel_boosted, cc900279]
[AFM_DEBUG] unboosted_order=[..., 288e689e, cd25f777, e25be2d7, cxgen:cxgen:9]
```

**Agent analysis** (excerpt):
> DV=control confirmed. `isExemptAfmCarousel` returns false immediately (early return), so no detail log fires. 4 NV carousels unpinned. AFM carousels visible but not pinned. Baseline behavior validated.

**Pass criteria**: 4/4 passed. Screenshots captured for each test showing carousel positions on homepage.

### Final Verdict (agent-produced)

| # | Assertion | Result |
|---|-----------|--------|
| 1 | T1: DV=control, afmExempt=false | PASS |
| 2 | T1: AFM carousels in unboosted_order | PASS |
| 3 | T2: DV=treatment, detection logic fires | PASS |
| 5 | T2: hasAfmVertical + hasAfmCampaign verified independently | PARTIAL (sandbox data limitation) |
| 9 | T3: Non-AFM NV still unpinned (regression) | PASS |

> **Overall: PASS WITH NOTES.** Core decision logic verified at every branch point. Full pinning behavior could not be triggered due to sandbox data composition, but code logic is sound. Recommendation: approve PR, verify pinning in production.

The agent identified a sandbox data limitation that would have been easy to miss manually. No carousel in the test environment had *both* the AFM vertical AND an active meal-box campaign simultaneously. This kind of nuanced analysis is exactly what makes the autonomous loop valuable.

Full results: `brain/afm-unpin-exemption-testing/results/`
