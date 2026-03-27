# Sandbox E2E — Architecture

**Last updated:** 2026-03-27

## Pipeline

```
Context            → Code Change        → Sandbox Environment    → Output Signals         → Results
(user request)       (diff synced via      (feed-service pod +      (app logs +              (report.md +
                      devbox run)           Playwright browser)      extraction data +        meta.json +
                                                                     Snowflake events)        PR evidence)

                              ┌──────────────────────────────────────┐
                              │         Orchestrator Agent           │
                              │   (drives entire pipeline E2E)      │
                              └──────────────────────────────────────┘
```

## Skill Composition

```
sandbox-run        ← top-level orchestrator (NOT YET BUILT)
├── sandbox-setup  ← provision/sync/teardown sandbox
├── sandbox-test   ← 3-load homepage test + audit trail
└── validate-iguazu← Snowflake event validation (optional)

feed-service-pr    ← PR lifecycle, invokes sandbox-test before create/push
```

Each skill works standalone. `sandbox-run` would compose them with auto-recovery.

## What sandbox-run Would Add

`sandbox-test` already handles: state reading, branch guard, pod liveness, audit trail, browser interaction, log capture, cross-load analysis, reporting.

`sandbox-run` uniquely adds:

| Responsibility | Description |
|---|---|
| Auto-invoke sandbox-setup | If state missing or pod dead → setup instead of stopping |
| Sync decision | `git status --porcelain` + commitHash comparison → conditional `devbox run` |
| Compose validate-iguazu | Chain Snowflake validation after browser test |
| Single entry point | "test this change" → figures out what's needed |

**Open decision:** Separate skill or fold into `sandbox-test`? See `spec/01-sandbox-run.md`.

## Audit Trail

```
feed-service/.claude/sandbox/runs/<run-id>/
  meta.json, steps.jsonl, feed-service.log,
  load-{1,2,3}-carousels.json, report.md, analyze.py,
  homepage-evidence.png, session.webm, iguazu-validation.md
```

Run ID: `<timestamp>_<branch-sanitized>_<short-hash>`

## Key Design Decisions

| Decision | Rationale |
|---|---|
| 3 independent loads per test | Detects non-determinism, ensures data integrity |
| Single-pass extraction | Scroll + discover + extract + click Next in one sweep. Efficient. |
| MutationObserver content settling | Replaces fixed waits. 200ms quiet threshold. |
| Video over screenshots | Session `.webm` captures everything. Single `homepage-evidence.png` for PR attachments. |
| No pruning | Runs accumulate. Address later if disk becomes an issue. |
| Per-load file isolation | `load-N-carousels.json` — never combine DOM data across loads |
