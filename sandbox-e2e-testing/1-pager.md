# Feed Service Homepage E2E Sandbox Testing — Vision

This document is the **aspirational end-state**. See `plan.md` for what's built and what's next.

## End-to-End Pipeline

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

## Assertion Layers

| Layer | Description | Status |
|---|---|---|
| Log / Network | feed-service logs, API response shape, no load errors | **Built** |
| Snowflake Events | impression events, ranking signals, item IDs & positions | **Built** |
| Ranking Sanity | carousel scores ordered correctly, store scores match position | Partial — scores extracted, no order assertion |
| Visual | screenshot diff against baseline, structural snapshot | Partial — screenshot + video, no baseline diff |
| DV / Experiment | enrollment confirmed, correct variant assigned | Not started |
| Shadow Traffic | prod sample routed to change branch, latency & response compared | Not started |
| Debug Analysis | PR diff instrumented with temp logs, variable values validated | Not started |

## Debug Agent — Two-Phase Flow

The most ambitious piece. Bridges PR diff → runtime observability. Instruments code before load, interprets output after.

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

    Note over DA,Logs: Homepage loads with debug instrumentation
    Browser->>Code: Navigate homepage
    Code->>Logs: Emit debug output alongside normal logs

    Note over DA,Logs: Phase 2 — Analysis (post-load)
    DA->>Logs: Read all captured debug output
    DA->>DA: Correlate values against expected behavior from diff
    DA->>DA: Flag unexpected nulls, wrong types, missing calls
    DA->>Code: Remove temp debug statements
```

## Target Report Format

```
┌─────────────────────────────────────────────────────────────────────────┐
│   Homepage E2E Test Report                                              │
│   Sandbox: user-abc  |  2026-03-18 14:32  |  run #42                    │
├─────────────────────────────────────────────────────────────────────────┤
│   🔍  Debug Analysis      ✅  4 vars checked — all values in range       │
│   📋  Feed Load & Logs    ✅  0 errors, 2.1s load                        │
│   🔀  Shadow Traffic      ✅  p50 +2ms vs baseline, responses match      │
│   📡  Snowflake Events    ✅  12 / 12 events emitted                     │
│   🔬  DV / Experiments    ✅  exp-ranking-v3 enrolled                    │
│   🖥️  Visual Snapshot     ⚠️  1 layout diff flagged                      │
│   🧪  Ranking Sanity      ❌  carousel[2] score/order mismatch           │
├─────────────────────────────────────────────────────────────────────────┤
│   Overall: FAIL (4/6)  —  1 warning, 1 failure                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## CI Mode (Not Yet Built)

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant PR as GitHub PR
    participant Pipeline as Test Pipeline
    participant Env as Sandbox Env

    Dev->>PR: opens pull request
    PR->>Pipeline: GitHub Actions trigger
    Pipeline->>Env: provision sandbox
    Pipeline->>Pipeline: run full assertion suite
    Pipeline-->>PR: post report as PR comment
```

## Open Questions
- How to parameterize DV/experiment overrides in sandbox?
- Visual diffs: opt-in or always-on?
- Debug Agent: can it reliably identify meaningful instrumentation points from a diff?
