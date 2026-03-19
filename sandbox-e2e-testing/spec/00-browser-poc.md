# Spec 00 — Browser POC (Prerequisite)

## Status
**MUST COMPLETE BEFORE PHASE 1 IMPLEMENTATION.**

IT dept manages Chrome — need to confirm Playwright MCP can control it before building any browser-dependent skills.

---

## Goal

Prove that Playwright MCP can:
1. Launch / connect to Chrome
2. Navigate to a URL
3. Scroll carousel elements
4. Take and return screenshots

---

## POC Steps

Run these manually via Playwright MCP in a Claude session:

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

- **Chrome not found**: Playwright can't locate IT-managed Chrome binary → need to pass explicit `executablePath`
- **SSO wall**: doordash.com redirects to login → use doordashtest.com instead (sandbox routing already handles this)
- **Managed browser restrictions**: Chrome policies blocking automation flags → may need `--remote-debugging-port` workaround or to use a non-managed browser profile
- **Playwright MCP not configured**: Check `~/.claude/settings.json` for Playwright MCP server entry

---

## Outcome

Document results here after running POC:

```
Date:
Chrome binary path:
Playwright MCP config:
Result: pass / fail
Blockers found:
Workaround applied:
```

---

## Implementation Dependency

`sandbox-run` and `sandbox-test` browser steps are **blocked** on this POC passing.
