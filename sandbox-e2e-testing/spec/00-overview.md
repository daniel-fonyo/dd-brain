# Spec Overview — Sandbox E2E

## Implementation Order

Implement in this order. Each spec is a prerequisite for the next.

1. `03-audit-trail.md` — define the audit structure first; everything else writes into it
2. `01-sandbox-run.md` — orchestrator skill; depends on audit trail contract
3. `02-sandbox-test-update.md` — update sandbox-test to write into runDir

Do not implement Phase 2+ specs until Phase 1 is working end-to-end.

---

## File Locations

| Artifact | Location |
|---|---|
| Skill files | `~/.claude/commands/*.md` |
| Sandbox state | `~/.claude/sandbox-state.json` |
| Audit runs | `feed-service/.claude/sandbox/runs/<run-id>/` |
| Latest run symlink | `feed-service/.claude/sandbox/latest` |
| Gitignore entry | `feed-service/.gitignore` |

---

## Skill Contract

Every skill must:
- Start by reading relevant state (sandbox-state.json, runDir)
- Write to steps.jsonl at each significant step
- Report progress inline to the user as it goes (not just at the end)
- Return a structured result the caller can act on

---

## Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Run ID | ISO timestamp + 6-char hex | `2026-03-19T14-30-00-a1b2c3` |
| Step names | kebab-case verbs | `pod-check`, `browser-navigate` |
| Screenshot files | zero-padded sequence + label | `01-homepage.png`, `02-carousel-stores.png` |
| Log file | `feed-service.log` | always this name inside runDir |
