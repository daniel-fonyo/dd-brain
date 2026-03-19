# Feed Service Homepage E2E Sandbox Testing — 1-Pager

## Visuals

### 1. End-to-End Pipeline


```mermaid
flowchart TD
    A([Developer / CI]) --> Orch
    PR([PR Diff]) --> Orch

    subgraph Orch["🤖 Orchestrator Agent"]
        O1[Decompose task, spawn subagents, collect results]
    end

    Orch --> DA1
    Orch --> BA
    Orch --> STA

    subgraph DA1["🔍 Debug Agent — Phase 1: Instrument"]
        D1[Analyze diff] --> D2[Identify log insertion points] --> D3[Inject temp debug statements]
    end

    DA1 --> BA

    subgraph BA["🌐 Browser Agent"]
        B1[Navigate homepage] --> B2[Feed loads] --> B3[Capture logs, network, screenshots]
    end

    subgraph STA["🔀 Shadow Traffic Agent"]
        S1[Sample prod traffic] --> S2[Route to change branch] --> S3[Compare vs baseline]
    end

    B3 --> DA2
    B3 --> AA
    S3 --> AA

    subgraph DA2["🔍 Debug Agent — Phase 2: Analyze"]
        DAN1[Read debug logs] --> DAN2[Validate variable values and state] --> DAN3[Flag unexpected values or flow]
    end


    subgraph AA["🧪 Assertion Agents"]
        AA1[Log Analysis Agent]
        AA2[Snowflake Event Agent]
        AA3[DV / Experiment Agent]
        AA4[Visual Inspection Agent]
        AA5[Ranking Sanity Agent]
    end

    DA2 --> RA
    AA --> RA

    subgraph RA["📊 Report Agent"]
        R1[Aggregate results, flag failures, format report]
    end

    RA --> Z[/Report/]
```

---

### 2. Assertion Layers

```mermaid
block-beta
  columns 1
  A["🖥️  Visual Layer — screenshot diff, structural snapshot, carousel interactions"]
  B["🧪  Ranking Sanity — carousel scores ordered correctly; item/store scores within each carousel match visual position"]
  C["🔬  DV / Experiment Layer — enrollment confirmed, correct variant assigned"]
  D["📡  Snowflake Event Layer — impression events, ranking signals, item IDs & positions"]
  E["📋  Log / Network Layer — feed-service logs, API response shape, no load errors"]
  F["🔀  Shadow Traffic Layer — prod sample routed to change branch; latency & response compared to baseline"]
  G["🔍  Debug Analysis Layer — PR diff instrumented with temp logs; variable values and execution flow validated post-load"]
```

---

### 3. Test Report Mockup

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   Homepage E2E Test Report                                              │
│   Sandbox: user-abc  |  2026-03-18 14:32  |  run #42                    │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   🔍  Debug Analysis      ✅  4 vars checked — all values in range       │
│   📋  Feed Load & Logs    ✅  0 errors, 2.1s load                        │
│   🔀  Shadow Traffic      ✅  p50 +2ms vs baseline, responses match      │
│   📡  Snowflake Events    ✅  12 / 12 events emitted                     │
│   🔬  DV / Experiments    ✅  exp-ranking-v3 enrolled                    │
│   🖥️  Visual Snapshot     ⚠️  1 layout diff flagged                      │
│   🧪  Ranking Sanity      ❌  carousel[2] score/order mismatch           │
│                               scores: A>B  |  shown: B>A                │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Overall: FAIL (4/6)  —  1 warning, 1 failure                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 4. Debug Agent — Two-Phase Flow

The Debug Agent bridges the gap between a PR diff and runtime observability. It instruments the code before load and interprets the output after — performing the same validation loop a human engineer would run manually.

```mermaid
sequenceDiagram
    participant DA as Debug Agent
    participant Code as feed-service
    participant Browser as Browser Agent
    participant Logs as Captured Logs

    Note over DA,Logs: Phase 1 — Instrumentation (pre-load)
    DA->>DA: Read PR diff
    DA->>DA: Identify changed functions, variables, call sites
    DA->>Code: Inject temp debug statements at critical points
    Note right of Code: log(rankingScore), log(blendWeights), log(experimentFlag)

    Note over DA,Logs: Homepage loads with debug instrumentation
    Browser->>Code: Navigate homepage
    Code->>Logs: Emit debug output alongside normal logs

    Note over DA,Logs: Phase 2 — Analysis (post-load)
    DA->>Logs: Read all captured debug output
    DA->>DA: Correlate values against expected behavior from diff
    DA->>DA: Validate variable values are in expected range
    DA->>DA: Validate execution flow hit expected paths
    DA->>DA: Flag unexpected nulls, wrong types, missing calls
    DA->>Code: Remove temp debug statements
    DA-->>DA: Emit debug analysis result to Report Agent
```

**Example Debug Agent output**
```
🔍 Debug Agent — Instrumenting from PR diff...

  Identified 4 insertion points:
  + ranker.go:142     log rankingScore before sort
  + blender.go:87     log blendWeights after normalization
  + experiments.go:31 log experimentFlag assignment
  + scorer.go:204     log itemScores after model inference

  Loading homepage...

  Analyzing debug output...
  ✅ rankingScore (ranker.go:142)     0.87, 0.72, 0.61 — in expected range
  ✅ blendWeights (blender.go:87)     [0.6, 0.3, 0.1] — normalized correctly
  ❌ experimentFlag (experiments.go:31) got nil — expected "ranking-v3"
       → experiment not enrolled at this point in execution
  ✅ itemScores (scorer.go:204)       shape [24] — matches expected carousel size

  Removing temp debug statements...
  Result: 1 failure — experimentFlag nil at assignment
```

---

### 5. Developer Feedback Loop


Three ways to trigger the same pipeline — manual handoff, on-demand report, and CI on every PR.

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant CLI as Claude CLI
    participant Skill as Sandbox Skill
    participant Env as Sandbox Env
    participant Pipeline as Test Pipeline
    participant PR as GitHub PR

    Note over Dev,PR: Mode 1 — Hand off sandbox setup
    Dev->>CLI: "set up my sandbox env"
    CLI->>Skill: invoke sandbox-setup
    Skill->>Env: provision / resync
    Env-->>CLI: ready
    CLI-->>Dev: Sandbox ready at localhost:3000

    Note over Dev,PR: Mode 2 — Sync changes and get report
    Dev->>CLI: "sync my latest changes and run assertions"
    CLI->>Skill: invoke sandbox-test (sync branch)
    Skill->>Env: pull branch changes
    Skill->>Pipeline: load homepage + run full assertion suite
    Pipeline-->>CLI: assertion results
    CLI-->>Dev: report inline in terminal

    Note over Dev,PR: Mode 3 — CI reports on every PR
    Dev->>PR: opens pull request
    PR->>Pipeline: GitHub Actions trigger
    Pipeline->>Env: provision sandbox
    Pipeline->>Pipeline: run full assertion suite
    Pipeline-->>PR: post report as PR comment
```

---

### 6. CLI Interaction Mockups

**Mode 1 — Sandbox setup**
```
$ claude "set up my sandbox env"

  Spinning up sandbox for user-abc...
  Syncing feed-service branch: feat/ranking-v3
  Sandbox ready at localhost:3000 ✅
```

**Mode 2 — Sync + assertion report**
```
$ claude "sync my latest changes and give me an assertion report"

  Pulling feat/ranking-v3...
  Loading homepage...
  Running assertions...

  ┌──────────────────────────────────────────────────┐
  │  Homepage E2E Report  |  feat/ranking-v3         │
  ├──────────────────────────────────────────────────┤
  │  Feed Load & Logs    ✅  0 errors, 2.1s          │
  │  Shadow Traffic      ✅  p50 +2ms vs baseline    │
  │  Snowflake Events    ✅  12 / 12 emitted         │
  │  DV / Experiments    ✅  exp-ranking-v3 enrolled │
  │  Visual Snapshot     ⚠️  1 layout diff flagged  │
  │  Ranking Sanity      ❌  carousel[2] score/order │
  │                          mismatch (A>B shown B>A)│
  ├──────────────────────────────────────────────────┤
  │  FAIL (4/6)  —  details: /tmp/e2e-report-42/    │
  └──────────────────────────────────────────────────┘
```

**Mode 3 — CI PR comment**
```
🤖 Homepage E2E Report — feat/ranking-v3

| Check            | Result  | Detail                          |
|------------------|---------|---------------------------------|
| Feed Load & Logs | ✅ PASS | 0 errors, 2.1s load             |
| Shadow Traffic   | ✅ PASS | p50 +2ms vs baseline            |
| Snowflake Events | ✅ PASS | 12/12 emitted                   |
| DV / Experiments | ✅ PASS | exp-ranking-v3 enrolled         |
| Visual Snapshot  | ⚠️ WARN | 1 layout diff — [view diff]()   |
| Ranking Sanity   | ❌ FAIL | carousel[2] score/order mismatch|

Overall: FAIL — 1 warning, 1 failure
```

---

## Problem

Validating homepage ranking changes today is manual and slow. Engineers spin up sandbox environments, load the homepage, eyeball the feed, check logs, and try to correlate events and experiment assignments by hand. This is error-prone, doesn't scale, and creates risk when shipping ranking changes.

## Goal

Automate end-to-end homepage validation in sandbox environments so engineers get fast, reproducible signal that:
- The feed loaded correctly
- The right DVs/experiments were applied
- Snowflake events were emitted as expected
- Visual ranking and surface appearance is correct

## Scope

### In Scope
- Spinning up sandbox environments on demand
- Reloading the homepage and triggering a full feed load cycle
- Asserting Snowflake events emitted on page load (impression events, ranking signals)
- Asserting DVs and experiment assignments are present and correct
- Visual inspection of homepage feed (ranking order, surface appearance)
- Basic interactions: scroll, tap/click cards, carousel navigation
- Log analysis: surface errors, unexpected ranking decisions, blending anomalies

### Out of Scope (for now)
- Multi-user / multi-session load testing
- Full regression coverage beyond homepage
- Non-sandbox environments (staging, prod)

## Approach

### 1. Sandbox Provisioning
- Reuse existing sandbox-setup skill/tooling to spin up or resync sandbox env
- Parameterize by user profile, DV overrides, experiment config

### 2. Homepage Load + Log Capture
- Automate homepage navigation (Playwright)
- Capture all network requests, console logs, server-side logs from feed-service
- Assert no load errors, expected response shape from feed ranking endpoint

### 3. Snowflake Event Validation
- After load, query Snowflake (or event sink) for expected impression/engagement events
- Assert event presence, correct item IDs, expected ranking positions

### 4. DV / Experiment Assertion
- Parse feed response or server logs for DV assignment payloads
- Assert expected experiments are enrolled, correct variant applied

### 5. Visual Inspection
- Screenshot homepage at load
- Playwright snapshot for accessibility/structural validation
- Optionally: image diff against baseline to catch visual regressions

### 6. Ranking Sanity Checks
- Assert top-N items match expected ranking heuristics (e.g. no obviously wrong content surfaced)
- Flag anomalies: duplicate content, wrong content type, ranking inversions

## Output / Reporting
- Pass/fail summary per assertion category
- Linked logs, screenshots, and Snowflake event traces for failures
- CI-compatible: can be triggered pre-merge or on a schedule

## Success Criteria
- An engineer can run a single command and get a green/red signal on homepage health
- False positive rate is low enough that failures are actionable
- Covers the most common ranking regression patterns caught manually today

## Open Questions
- What's the right event sink to query — Snowflake directly, or an intermediate event store?
- How do we parameterize DV/experiment overrides in sandbox?
- Should visual diffs be opt-in or always-on?
- What latency is acceptable for the full test run?

## Next Steps
1. Audit existing sandbox-setup and sandbox-test skills for reuse
2. Define the minimal set of Snowflake events to assert
3. Prototype Playwright-based homepage load + log capture
4. Define DV/experiment assertion contract with feed-service response schema
