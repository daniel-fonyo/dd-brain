# Sandbox E2E — Implementation Plan

## Context
Automating "test this change on homepage" — a single command that figures out sandbox state, syncs changes, interacts with the homepage, captures all observability, and reports back.

Reference: `1-pager.md` (vision), `design.md` (architecture).

---

## Phases

### Phase 1 — Orchestrator + Audit Trail (NOW)
Get the core loop working end-to-end with audit persistence.

| Step | Deliverable | Spec | Status |
|---|---|---|---|
| 1.1 | `sandbox-run.md` skill | `spec/01-sandbox-run.md` | pending |
| 1.2 | Update `sandbox-test.md` — runDir + structured output | `spec/02-sandbox-test-update.md` | pending |
| 1.3 | Audit trail format + gitignore | `spec/03-audit-trail.md` | pending |

**Done when:** `"test this change on homepage"` → sandbox-run orchestrates full flow end-to-end with audit written to `feed-service/.claude/sandbox/runs/`.

---

### Phase 2 — Smarter Sync + Log Analysis
Make the sync and log layers more useful.

| Step | Deliverable | Spec | Status |
|---|---|---|---|
| 2.1 | Structured log parsing (ERROR/WARN extraction, request trace) | `spec/04-log-analysis.md` | pending |
| 2.2 | `devbox run` lifecycle management (keep-alive vs re-sync detection) | `spec/05-devbox-lifecycle.md` | pending |
| 2.3 | `report.md` standardized template | `spec/03-audit-trail.md` | pending |

---

### Phase 3 — Assertion Layers (from 1-pager)
Layered signal beyond "did it load."

| Step | Deliverable | Spec | Status |
|---|---|---|---|
| 3.1 | Ranking sanity — carousel score order validation | `spec/06-ranking-sanity.md` | pending |
| 3.2 | DV / experiment enrollment assertion | `spec/07-dv-experiment.md` | pending |
| 3.3 | Snowflake event assertion | `spec/08-snowflake-events.md` | pending |

---

### Phase 4 — CI Integration
Trigger the full pipeline from GitHub Actions on every PR.

| Step | Deliverable | Spec | Status |
|---|---|---|---|
| 4.1 | GitHub Actions workflow | `spec/09-ci.md` | pending |
| 4.2 | PR comment reporter | `spec/09-ci.md` | pending |

---

## Guiding Principles
- Each skill works standalone — sandbox-run composes, not replaces
- Audit-first: write steps.jsonl as work happens, not after
- Fail fast: pod liveness check before any test attempt
- Evidence always captured: logs + screenshots per run, never overwritten
- Natural language trigger: user says what they want, skill figures out state

---

## Current Focus
**Phase 1** — implement `sandbox-run`, update `sandbox-test`, scaffold audit trail.
