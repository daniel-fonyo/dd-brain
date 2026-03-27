# Spec Overview — Sandbox E2E

**Last updated:** 2026-03-27

## Specs

| Spec | Purpose | Status |
|---|---|---|
| `00-browser-playbook.md` | Living reference: deterministic browser interaction sequence | Active |
| `01-sandbox-run.md` | Orchestrator skill design | Not started |
| `03-audit-trail.md` | Audit trail contract (directory structure, JSON schemas) | Reference |

## File Locations

| Artifact | Location |
|---|---|
| Skill files | `~/.claude/commands/*.md` |
| Sandbox state | `~/.claude/sandbox-state.json` |
| Playwright MCP config | `~/.claude/playwright-mcp-config.json` |
| Audit runs | `feed-service/.claude/sandbox/runs/<run-id>/` |
| Latest run symlink | `feed-service/.claude/sandbox/runs/latest` |

## Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Run ID | `<timestamp>_<branch-sanitized>_<short-hash>` | `2026-03-19T14-30-00_feat-dfonyo-ranking-debug_a1b2c3d` |
| Step names | kebab-case verbs | `pod-check`, `browser-navigate` |
| Per-load data | `load-<N>-carousels.json` | `load-1-carousels.json` |
| Evidence screenshot | `homepage-evidence.png` | always this name inside runDir |
| Log file | `feed-service.log` | always this name inside runDir |
| Video file | `session.webm` | always this name inside runDir |
