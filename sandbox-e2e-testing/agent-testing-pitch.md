# Autonomous Agent-Driven Testing for Feed-Service Homepage

## The Problem

Homepage development in feed-service is bottlenecked by manual testing. Every change — ranking adjustments, experiment rollouts, new carousel types, blending logic — requires an engineer to:

1. Spin up a sandbox environment
2. Load the homepage in a browser and visually inspect the feed
3. Cross-reference pod logs to verify ranking decisions
4. Check Snowflake events to confirm analytics fired correctly
5. Verify experiment enrollment and DV assignments
6. Repeat for different scenarios and edge cases

This is slow, error-prone, and doesn't scale. A single change can take 30-60 minutes of manual validation. The homepage is a black box — ranking inputs go in, a feed comes out, and verifying that the feed is *correct* requires correlating signals across multiple systems (service logs, browser rendering, Snowflake events, experiment platforms) that were never designed to be observed together.

**The result:** We can't match the pace at which the business needs homepage changes shipped. Requests pile up, testing becomes the bottleneck, and the risk of regression grows with every change because thorough validation takes too long to do every time.

<!-- TODO: Add specific pain points — recent incidents, turnaround times, backlog size -->

## Why This Matters Now

The volume of homepage-related asks is increasing — new verticals, ranking experiments, UBP migration, blending changes. If we try to absorb this workload with the current manual workflow, one of two things happens:

1. **We slow down** — testing becomes the bottleneck and delivery timelines stretch
2. **We cut corners** — changes ship with less validation and regressions slip through

Neither is acceptable. We need to invest in our development workflow *first* to unlock the throughput required to handle what's coming.

## The Solution: Autonomous Agent-Driven Sandbox Testing

Instead of manually testing every change, we use a Claude Code agent that has the tools to autonomously validate homepage changes end-to-end. The engineer describes *what* to test. The agent handles *how*.

### How It Works

```
                    ┌──────────────────────────────────────────────────────┐
                    │                                                      │
       ┌────────────▼───────────┐                                         │
       │   Testing Plan         │                                         │
       │   (engineer + agent    │                                         │
       │    collaborate on      │                                         │
       │    what to validate)   │                                         │
       └────────────┬───────────┘                                         │
                    │                                                      │
       ┌────────────▼───────────┐         ┌───────────────────────┐       │
       │   Agent Decision       │────────▶│   Sandbox System      │       │
       │   (what input to       │         │   (feed-service pod   │       │
       │    provide next)       │         │    + live browser)    │       │
       └────────────────────────┘         └───────────┬───────────┘       │
                    ▲                                  │                   │
                    │                                  ▼                   │
       ┌────────────┴───────────┐         ┌───────────────────────┐       │
       │   Analysis             │◀────────│   Output Signals      │       │
       │   (compare outputs     │         │   - Application logs  │       │
       │    against plan        │         │   - Browser rendering │       │
       │    success criteria)   │         │   - Snowflake events  │       │
       └────────────┬───────────┘         └───────────────────────┘       │
                    │                                                      │
                    └──────────────────────────────────────────────────────┘
                              ▲  Autonomous loop until
                              │  plan criteria are met
                              ▼
                    ┌───────────────────────┐
                    │   Final Output        │
                    │   - PR with results   │
                    │   - Video evidence    │
                    │   - Log analysis      │
                    │   - Snowflake queries │
                    │   - Pass/fail verdict │
                    └───────────────────────┘
```

The core is a **reinforcing flywheel**: the agent makes a decision (e.g., load the homepage, scroll to a carousel, enable debug mode), the system reacts, the agent observes the output (logs, rendered UI, Snowflake events), analyzes it against the testing plan's success criteria, and decides what to do next. This loop runs autonomously until all criteria are evaluated.

### What the Agent Can Do Today

| Capability | Description |
|---|---|
| **Sandbox lifecycle** | Spin up, sync code, tear down sandbox environments |
| **Browser interaction** | Navigate homepage, scroll, click through carousels, enable debug overlays — all via Playwright |
| **Log capture & analysis** | Stream pod logs in background, parse errors/warnings, correlate to specific requests |
| **ML score extraction** | Read ranking scores, second-pass scores, store-level scores from debug overlays |
| **Snowflake validation** | Query Iguazu events, 1:1 match browser observations to analytics records |
| **Multi-load testing** | Run 3 independent homepage loads, compare across loads for consistency |
| **PR evidence packaging** | Bundle screenshots, video, logs, queries into PR test sections |

### What This Gives Us

The agent produces a **full end-to-end validation report** — not a unit test pass/fail, but evidence that the homepage *actually works as expected* when a real browser hits it. This includes:

- Session video of the browser interacting with the homepage
- Structured data: every carousel, every store, every ML score extracted
- Cross-load stability analysis (do results change between loads?)
- Pod log analysis with error/warning breakdown
- Snowflake event correlation (did analytics fire correctly?)
- Pass/fail verdict with reasoning

<!-- TODO: Insert sample output from a recent agent test run -->

This mimics exactly what an engineer does manually — but faster, more thorough, and reproducible. The agent can observe signals that are impractical to check by hand every time: whether ranking scores are monotonically decreasing within a carousel, whether the same stores appear across multiple loads, whether every carousel in the browser has a corresponding Snowflake event.

## Why Now

1. **The tools exist.** Sandbox provisioning, browser automation (Playwright), Snowflake MCP, and the agent skills are all built and working. This is not a proposal to build from scratch — it's a proposal to invest in what we already have.

2. **Company alignment.** Anthropic's Claude Code and agentic workflows are a company-wide investment. We'd be applying these tools to a real engineering bottleneck, not experimenting in a vacuum.

3. **The alternative is worse.** Without this, every new homepage feature or ranking change carries the same manual testing overhead. As the pace of asks increases, this becomes the constraint that limits everything else.

## What We Need

- **Engineering time** to formalize the agent testing workflow and close remaining gaps (orchestrator skill, DV/experiment assertions, CI integration)
- **Team adoption** — engineers start using the agent for their sandbox testing, feeding back improvements
- **Iteration** — the flywheel improves with use. Each test run surfaces new signals the agent can check, new self-healing patterns for UI changes, and new assertion types

## Expected Impact

| Metric | Today (manual) | With agent testing |
|---|---|---|
| Time to validate a homepage change | 30-60 min per change | ~5 min (agent runs autonomously) |
| Confidence in validation | Variable — depends on how thorough the engineer is | Consistent — same checks every time, multi-load |
| Regression detection | Caught in review or production | Caught before PR merge |
| Evidence trail | Screenshots in Slack, verbal "looks good" | Structured report with video, logs, queries, scores |
| Scalability | Linear with engineer time | Parallel — agent can test while engineer works on next change |

---

<!-- TODO: Questions for Daniel to answer before sharing:
1. Can you provide 2-3 specific recent examples of painful manual testing cycles? (turnaround time, regressions caught late, etc.)
2. Do you have a sample agent test output to embed as evidence?
3. Who is the target audience — direct manager, skip-level, cross-team? (affects framing)
4. Is there a specific ask — headcount, dedicated sprint time, formal project status? Or is this awareness/buy-in first?
5. Should this reference the UBP migration timeline as a forcing function?
6. Any sensitive details to avoid (team names, specific incident postmortems)?
-->
