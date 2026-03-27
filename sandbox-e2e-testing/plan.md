# Sandbox E2E — Remaining Work

**Last updated:** 2026-03-27

## What's Built

| Skill | What it does |
|---|---|
| `sandbox-setup` | Provision/sync/teardown sandbox lifecycle |
| `sandbox-test` | 3-load homepage test, carousel extraction, log capture, cross-load analysis, audit trail |
| `validate-iguazu` | Snowflake event 1:1 validation against browser observations |
| `feed-service-pr` | PR lifecycle with mandatory sandbox testing + evidence packaging |

---

## What's Next

### 1. Orchestrator — `sandbox-run` (spec: `spec/01-sandbox-run.md`)

Single command for "test this change on homepage." Composes existing skills with auto-recovery.

**What it adds over running `/sandbox-test` directly:**
- Auto-invoke `/sandbox-setup` when no state or pod dead (instead of stopping)
- Devbox sync decision (`git status` → conditional `devbox run`)
- Optional chaining of `/validate-iguazu` after browser test
- Natural language entry point — user doesn't need to know which skill to invoke

**Open decision:** Build as separate skill, or fold auto-setup + sync into `sandbox-test`?

### 2. DV / Experiment Assertion

Parse feed response or server logs for DV assignment payloads. Assert expected experiments enrolled, correct variant applied. No spec written yet.

### 3. Ranking Order Assertion

Scores are already extracted per carousel and per store. Missing: explicit assertion that store ranking scores are monotonically decreasing in display order within each carousel. Cross-load comparison already exists.

### 4. Visual Baseline Diff

Screenshot + video captured per run. Missing: image diff against a baseline to detect visual regressions. Opt-in or always-on TBD.

### 5. CI Integration (GitHub Actions)

`feed-service-pr` runs sandbox-test before PR create/push, but there's no GitHub Actions workflow to run automatically on every PR. Needs: workflow definition, sandbox provisioning in CI, PR comment reporter.

### 6. Debug Agent (Aspirational)

Automated code instrumentation based on PR diffs — inject debug logging at changed call sites, interpret output after homepage load, flag unexpected values. Most ambitious piece, no spec yet.

### 7. Shadow Traffic (Aspirational)

Sample prod traffic, route to change branch, compare responses vs baseline. No spec yet.
