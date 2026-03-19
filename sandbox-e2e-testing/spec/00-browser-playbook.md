# Browser Interaction Playbook — doordashtest.com Homepage

Deterministic step sequence for sandbox-test browser interactions. Follow this known path by default. Only fall back to exploratory page analysis if a step fails or the UI has known breaking changes.

## Credentials

- **Domain**: `https://www.doordashtest.com/`
- **Email**: `tas-cx-doortest-egsn7vrtcl@doordash.com`
- **Password**: `1XQlCDr8Qb`
- **Note**: Test account on doortest tenant. Never use `doordash.com` — creds won't work there.

---

## Video Recording

The entire browser session is recorded automatically — no manual screenshot steps needed.

**Config** (`~/.claude/mcp.json`):
```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": [
        "@playwright/mcp@latest",
        "--viewport-size=1280x720",
        "--save-video=1280x720",
        "--output-dir=/tmp/playwright-mcp-output"
      ]
    }
  }
}
```

- **Format**: `.webm` (VP8) — GitHub-compatible for upload to PRs/issues
- **Resolution**: 1280x720 — readable and compact
- **Output**: `/tmp/playwright-mcp-output/` (staging) — move to audit trail after session
- Video is written when the browser context closes (`browser_close` or session end)

**Post-session**: copy video to audit trail:
```
cp /tmp/playwright-mcp-output/*.webm <runDir>/session.webm
```

---

## Step Sequence

### 1. Navigate with Latency Measurement

```
browser_evaluate → (() => { window.__navStart = Date.now(); })()
browser_navigate → https://www.doordashtest.com/
browser_evaluate → (() => { return Date.now() - window.__navStart; })()
```

Record result as `homepageLoadMs`. Fallback if variable lost (page reload clears it):
```
browser_evaluate → (() => {
  const nav = performance.getEntriesByType('navigation')[0];
  return nav ? Math.round(nav.loadEventEnd - nav.startTime) : null;
})()
```

### 2. Sign In (conditional — only if not authenticated)

Check: if page shows "Sign In" link → proceed with login. If already on `/home` with carousels → skip to step 3.

```
click → "Sign In" link (testId: signInButton)
  ↓ redirects to identity.doordash.com/auth full-page login
type → Email textbox: tas-cx-doortest-egsn7vrtcl@doordash.com
type → Password textbox: 1XQlCDr8Qb
click → "Continue to Sign In" button
  ↓ transitions to "Welcome back" / password confirmation
  ↓ password field may be pre-filled from previous step
  ↓ if empty, re-fill password
click → "Sign In" button
  ↓ "Do not refresh the page while we log you in..."
  ↓ redirects to doordashtest.com/home
wait → 5 seconds for homepage to fully render
```

**Key gotcha**: The login is two-step. After entering email+password and clicking "Continue to Sign In", the form transitions to a password-only confirmation. The password field may be cleared during this transition — always verify it's filled before clicking the final "Sign In".

### 3. Dismiss "How Fees Work" modal

```
click → "Got It" button (dialog: "How Fees Work")
```
- This modal appears on first visit after login. If not present, skip.

### 4. Enable Debug Mode

Activate the debug overlay **before** analyzing carousels — this exposes carousel IDs and ML ranking scores.

```
evaluate → document.querySelector('.ToggleContainer-sc-ba2scp-0 input[type="checkbox"]').click()
  ↓ debug overlays appear on all page components
```

- **Toggle selector**: `.ToggleContainer-sc-ba2scp-0 input[type="checkbox"]` (bottom-right corner)
- **Fallback**: If class name changes, look for a checkbox near "Debug Mode" text at bottom-right of page
- Verify: `PAGE LEVEL LOG` text appears in snapshot

### 5. Enumerate and Analyze Every Carousel

Starting from the top of the page, work through **every** store carousel.

**For each carousel:**

1. **Read metadata** from snapshot + debug overlay:
   - Carousel name (heading text, e.g., "Under $1 delivery fee")
   - Carousel ID from overlay: `carousel.standard:store_carousel:<id>`

2. **Read bottom bar** (persistent, updates per carousel section):
   - Ranking First Pass Score (usually `N/A` on homepage)
   - Ranking Second Pass Score (float)

3. **Scroll through all pages** of the carousel:
   - On each visible page, read every store card:
     - Store name
     - Store ID from overlay: `card.store:store:<id>`
     - Store Ranking Score (float from overlay)
     - Rating, delivery fee, delivery time if visible
   - Click "Next button of carousel" to advance
   - Repeat until Next is disabled (end of carousel)

4. **Stop condition**: Stop when reaching content that is NOT a store carousel — individual store tiles/grid, footer, or end of page.

### 6. Console Errors

```
browser_console_messages → level: error
```

Record count and content of any JS errors.

### 7. Close Browser

```
browser_close
```
- Finalizes session video to `/tmp/playwright-mcp-output/`
- Copy to audit trail: `cp /tmp/playwright-mcp-output/*.webm <runDir>/session.webm`

---

## Debug Overlay Reference

| Element | Overlay Format | Example |
|---|---|---|
| Carousel | `carousel.standard:store_carousel:<id>` | `carousel.standard:store_carousel:under_1_delivery` |
| Store card | `card.store:store:<store_id>` | `card.store:store:297366` |
| Banner | `banner:<type>-<id>` | `banner:dashpass-FTBanner-02` |
| Container | `container:<name>` | `container:doordash_reminder` |

**Per-store-card data:**
- **Store Ranking Score**: float (e.g., `0.0006`) — individual store relevance
- **Store Debug Tools** button — links to MX Tools, Search Diagnostic/Evaluation Tools

**Bottom bar (persistent):**
- **Ranking First Pass Score**: initial retrieval score (N/A on homepage, used in search)
- **Ranking Second Pass Score**: re-ranking score after all features applied

---

## Failure Recovery

| Failure | Recovery |
|---|---|
| "Sign In" not found | Page may already be authenticated — check URL for `/home` |
| Login returns error | Re-check password field is filled on second step |
| "How Fees Work" not found | Skip — only appears on first visit |
| Carousel Next disabled | End of carousel — move to next one |
| Debug Mode toggle not found | Class name may have changed — search for `input[type="checkbox"]` near bottom-right |
| Debug overlays not appearing | Toggle may not have fired — verify checkbox `checked` state via evaluate |
| Page shows blank/spinner | Wait longer (10s), then retry navigate |
| Video not saved | Ensure `browser_close` is called — video writes on context close |
| Video too large | Reduce viewport: `--save-video=800x600` in mcp.json |
| Latency measurement returns null | Use `performance.getEntriesByType('navigation')` fallback |
