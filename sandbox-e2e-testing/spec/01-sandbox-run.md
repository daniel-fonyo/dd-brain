# Spec 01 — sandbox-run skill

## File
`~/.claude/commands/sandbox-run.md`

## Purpose
Top-level orchestrator. Entry point for "test this change on homepage." Composes `sandbox-setup` + `sandbox-test` + optionally `validate-iguazu` into a single command with **auto-recovery** — instead of stopping when sandbox is dead, it fixes the problem and continues.

## What It Adds Over `/sandbox-test` Directly

| Responsibility | Description |
|---|---|
| Auto-invoke sandbox-setup | State missing or pod dead → setup instead of stopping |
| Sync decision | `git status --porcelain` + commitHash comparison → conditional `devbox run` |
| Compose validate-iguazu | Chain Snowflake validation after browser test |
| Single entry point | "test this change" → figures out what's needed |

## Open Decision

**Option A: Separate skill** — cleaner separation, easier to extend. Con: another file to maintain.

**Option B: Fold into sandbox-test** — fewer moving parts. Con: sandbox-test is already 650+ lines.

**Recommendation:** Option A.

## Flow

```
1. Read ~/.claude/sandbox-state.json
   └── missing → invoke /sandbox-setup, re-read

2. Verify branch
   └── user supplied branch? → checkout
   └── else → use current

3. Pod liveness check
   └── Running → proceed
   └── Dead → invoke /sandbox-setup restart, re-read

4. Sync decision
   └── git status has output OR commitHash differs → devbox run web-group1-remote
   └── else → skip

5. Invoke /sandbox-test

6. (Optional) Invoke /validate-iguazu

7. Report combined results
```
