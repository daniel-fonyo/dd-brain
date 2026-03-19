# Spec 00 — Browser POC (Prerequisite)

## Status
**MUST COMPLETE BEFORE PHASE 1 IMPLEMENTATION.**

IT dept manages Chrome — need to confirm Playwright MCP can control it before building any browser-dependent skills. Uses bundled Chromium (not IT-managed Chrome) — no policy restrictions expected.

---

## Setup

Playwright MCP is configured machine-level in `feed-service/.mcp.json` (gitignored). It does **not** live in brain or any checked-in location.

**Pre-requisites:**
1. Install Node: `brew install node`
2. `feed-service/.mcp.json` already created with:
   ```json
   {
     "mcpServers": {
       "playwright": {
         "command": "npx",
         "args": ["@playwright/mcp@latest"]
       }
     }
   }
   ```
3. Open a **Claude Code session from `feed-service/`** — Playwright MCP will be available as a tool after approving it on first use.

> **Note:** Playwright MCP uses bundled Chromium by default, not the IT-managed Chrome binary. This avoids managed browser policy restrictions.

---

## Goal

Prove that Playwright MCP can:
1. Launch bundled Chromium
2. Navigate to a URL
3. Scroll carousel elements
4. Take and return screenshots

---

## POC Steps

Run in a Claude Code session from `feed-service/`:

1. Navigate to `https://www.doordash.com/`
2. Wait for page to fully load (network idle)
3. Take a full-page screenshot — confirm it returns a visible image
4. Find any horizontally scrollable carousel on the page
5. Scroll it left and right (arrow clicks or drag)
6. Take a screenshot of the scrolled state

---

## Success Criteria

| Check | Success |
|---|---|
| Page loads | No navigation error |
| Screenshot returns | Image visible, not blank |
| Carousels found | At least one scrollable element located |
| Scroll works | Screenshot shows different carousel state after scroll |

---

## Failure Modes to Investigate

- **npx not found**: Node not installed → `brew install node`
- **Playwright MCP not loading**: Restart Claude Code session from feed-service after creating `.mcp.json`
- **SSO wall**: doordash.com redirects to login → switch to `doordashtest.com` (sandbox routing handles this for real tests)
- **Bundled Chromium download blocked**: Corporate network blocks Chromium download → may need to manually install via `npx playwright install chromium`

---

## Outcome

Document results here after running POC:

```
Date:
Node version:
Result: pass / fail
Blockers found:
Workaround applied:
```

---

## Implementation Dependency

`sandbox-run` and `sandbox-test` browser steps are **blocked** on this POC passing.
