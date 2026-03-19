# Browser Interaction Playbook — doordashtest.com Homepage

Deterministic step sequence for sandbox-test browser interactions. Follow this known path by default. When a step fails, use the self-healing protocol to discover the new correct approach, then update this playbook so the fix persists.

## Credentials

- **Domain**: `https://www.doordashtest.com/`
- **Email**: `tas-cx-doortest-egsn7vrtcl@doordash.com`
- **Password**: `1XQlCDr8Qb`
- **Note**: Test account on doortest tenant. Never use `doordash.com` — creds won't work there.

---

## Self-Healing Protocol

The doordashtest.com UI will evolve over time. When any step fails:

1. **Detect**: The step returns an unexpected result (selector not found, element missing, wrong page state)
2. **Diagnose**: Use `browser_evaluate` to inspect the current page state — what's actually on the page?
3. **Adapt**: Try alternative selectors/approaches to achieve the same goal
4. **Verify**: Confirm the adapted approach achieved the intended outcome
5. **Persist**: Update this playbook AND `~/.claude/commands/sandbox-test.md` with the corrected approach, noting the date and what changed

**Example**: If `.ToggleContainer-sc-ba2scp-0` no longer matches the Debug Mode toggle:
- Fallback: search for any `input[type="checkbox"]` near bottom-right of viewport
- If that works, update the primary selector in both files
- Add the old selector to a "Previously known selectors" note for reference

**Rules:**
- Never silently skip a failed step — always attempt recovery
- Log the failure and recovery in the test report
- After a successful run with corrections, update both the playbook and the skill definition
- Keep a changelog at the bottom of this file for tracking UI changes

---

## Video Recording

The entire browser session is recorded automatically — no manual screenshot steps needed.

**Mechanism**: Playwright's `contextOptions.recordVideo` in a dedicated config file. The MCP server is started with `--config` pointing to this file.

**Config file** (`~/.claude/playwright-mcp-config.json`):
```json
{
  "browser": {
    "browserName": "chromium",
    "launchOptions": { "headless": true },
    "contextOptions": {
      "viewport": { "width": 1280, "height": 720 },
      "recordVideo": {
        "dir": "/tmp/playwright-mcp-output",
        "size": { "width": 1280, "height": 720 }
      }
    }
  }
}
```

**MCP server config** (`~/.claude.json` and `~/.claude/mcp.json`):
```json
{
  "args": ["x", "@playwright/mcp@latest", "--config", "/Users/daniel.fonyo/.claude/playwright-mcp-config.json"],
  "command": "/Users/daniel.fonyo/.devbox/ai/claude/bun"
}
```

- **Format**: `.webm` (VP8) — GitHub-compatible for upload to PRs/issues
- **Resolution**: 1280x720 — readable and compact
- **Output**: `/tmp/playwright-mcp-output/<uuid>.webm` — each session gets a random UUID filename
- Video is finalized when `browser_close` is called — not before
- **Important**: MCP config changes require a Claude Code session restart to take effect
- **Note**: `--save-video` is NOT a valid `@playwright/mcp` flag. Use `contextOptions.recordVideo` in the config file instead.

**Post-session**: find and copy video to audit trail, then create 2x speed version:
```bash
VIDEO_FILE=$(ls -t /tmp/playwright-mcp-output/*.webm 2>/dev/null | head -1)
cp "${VIDEO_FILE}" <runDir>/session.webm

# 2x speed version for faster review
ffmpeg -i "<runDir>/session.webm" -filter:v "setpts=0.5*PTS" -an "<runDir>/session-2x.webm"
```

- `session.webm` — original speed recording
- `session-2x.webm` — 2x speed for quick review
- Requires `ffmpeg` (`brew install ffmpeg`). If unavailable, skip 2x version and note in report.

---

## Critical: Snapshot Token Limits with Debug Mode

When Debug Mode is ON, page snapshots exceed 700K+ characters, blowing token limits.

- **Use `browser_evaluate`** for all interactions after Debug Mode is enabled — NOT `browser_click` or `browser_snapshot`
- **Use `browser_run_code`** for complex DOM extraction — it runs in a separate Playwright context
- **Never call `browser_snapshot`** after Debug Mode is ON — the accessibility tree is too large
- Modal dismissal, Debug Mode toggle, and carousel extraction all use `evaluate`-based approaches

---

## Step Sequence

### 0. Pod Liveness Check (REQUIRED)

**Hard gate — do NOT proceed without a live pod.** Without a live sandbox pod, traffic goes to the default doordashtest.com feed, not our code changes.

```bash
kubectl -n feed-service-sandbox get pod <pod> --no-headers 2>&1
```

- **Running** → proceed
- **NotFound / Error** → sandbox has expired. Do NOT test — results would reflect default feed, not our changes.

**Why**: `doordashtest.com` always serves content. Without sandbox routing to a live pod, the page renders the default production-like feed. The test would succeed but data wouldn't reflect code changes at all.

### 1. Navigate with Latency Measurement

#### 1a. Activate sandbox routing
Navigate to `homepageUrl` (the `doordashtest.com/developer/sandbox/...` URL). Wait for navigation to fully settle. This sets sandbox routing cookies.

#### 1b. Measure homepage load latency
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

**Proven flow** (tested 2026-03-19, identity.doordash.com):
```
browser_snapshot → check for "Sign In" link
  if not found → already authenticated, skip to step 3

browser_click → ref for "Sign In" link (testId: signInButton)
  ↓ redirects to identity.doordash.com/auth

browser_fill_form → [
  { name: "Email", type: "textbox", ref: <email field ref>, value: "tas-cx-doortest-egsn7vrtcl@doordash.com" },
  { name: "Password", type: "textbox", ref: <password field ref>, value: "1XQlCDr8Qb" }
]

browser_click → "Continue to Sign In" button
  ↓ transitions to "Welcome back" password confirmation step
  ↓ snapshot will show password field (may or may not be pre-filled)

  if password field is empty → browser_type into password field: "1XQlCDr8Qb"

browser_click → "Sign In" button
  ↓ "Do not refresh the page while we log you in..."
  ↓ redirects to doordashtest.com/home

browser_wait_for → time: 5  (wait for homepage to fully render)
```

**Key facts:**
- Login is **TWO-STEP**: email+password → "Continue to Sign In" → password confirmation → "Sign In"
- Password field **MAY be cleared** during step transition — always check before final click
- The identity page uses `browser_fill_form` (not type) — fill both fields at once
- After "Sign In", redirect takes 2-3 seconds — wait 5 seconds for homepage to render

**Self-heal targets**: If login flow changes, look for: form with email/password fields on `identity.doordash.com`, any button containing "Sign In" or "Log In" text, redirect back to `doordashtest.com/home`.

### 3. Dismiss "How Fees Work" modal

```
browser_evaluate → (() => {
  const buttons = [...document.querySelectorAll('button')];
  const gotIt = buttons.find(b => b.textContent.trim() === 'Got It');
  if (gotIt) { gotIt.click(); return 'dismissed'; }
  return 'no modal';
})()
```

- Use `evaluate` instead of snapshot+click — avoids snapshot overhead
- This modal appears on first visit after login. If not present, skip.

**Self-heal**: If button text changes, search for any modal/dialog with a single dismissal button.

### 3b. Inject Click Indicator (Visual Feedback for Video)

Inject a click event listener that creates a neon green pulsing circle at every click location. Makes the session video much more useful.

```
browser_evaluate → (() => {
  document.addEventListener('click', (e) => {
    const circle = document.createElement('div');
    circle.style.cssText = `
      position: fixed; left: ${e.clientX - 20}px; top: ${e.clientY - 20}px;
      width: 40px; height: 40px; border-radius: 50%;
      border: 3px solid #39FF14; background: rgba(57,255,20,0.3);
      pointer-events: none; z-index: 999999;
      animation: clickPulse 0.6s ease-out forwards;
    `;
    document.body.appendChild(circle);
    setTimeout(() => circle.remove(), 700);
  }, true);

  const style = document.createElement('style');
  style.textContent = `
    @keyframes clickPulse {
      0% { transform: scale(0.5); opacity: 1; }
      100% { transform: scale(2); opacity: 0; }
    }
  `;
  document.head.appendChild(style);
  return 'click indicator injected';
})()
```

- Inject after sign-in and modal dismissal, before debug mode toggle
- Uses `capture: true` so it fires before any `stopPropagation` calls
- Re-inject if a full page navigation occurs (unlikely after homepage load)

### 4. Enable Debug Mode

Activate the debug overlay **before** analyzing carousels — this exposes carousel IDs and ML ranking scores.

**Primary selector** (as of 2026-03-19):
```
browser_evaluate → (() => {
  const cb = document.querySelector('.ToggleContainer-sc-ba2scp-0 input[type="checkbox"]');
  if (cb) { cb.click(); return 'toggled'; }
  return 'not found';
})()
```

**Verify it worked:**
```
browser_evaluate → (() => document.body.innerText.includes('PAGE LEVEL LOG'))()
```

**Fallback** (if primary selector fails):
```
browser_evaluate → (() => {
  const checkboxes = [...document.querySelectorAll('input[type="checkbox"]')];
  const bottomRight = checkboxes.find(cb => {
    const r = cb.getBoundingClientRect();
    return r.y > window.innerHeight - 100 && r.x > window.innerWidth - 200;
  });
  if (bottomRight) { bottomRight.click(); return 'toggled via fallback'; }
  return 'not found';
})()
```

**Key fact:** Debug Mode is a checkbox inside `ToggleContainer-sc-ba2scp-0`, NOT a button. Use `evaluate` to click it — do NOT use `browser_click` with snapshot ref, as the massive debug-mode snapshot will exceed token limits.

**Self-heal**: The styled-component class hash (`ba2scp-0`) WILL change on builds. The fallback (checkbox near bottom-right) is position-based and more resilient. If even that fails, search for any element containing "Debug" or "Debug Mode" text and look for a nearby toggle/checkbox.

### 5. Enumerate and Analyze Every Carousel (Two-Phase)

This is the core analysis step. Use `browser_run_code` with the two-phase extraction approach. Do NOT try to parse the accessibility snapshot for carousel data — it's too large with debug mode on.

**Phase 1**: Vertical scroll to discover all carousel positions and IDs
**Phase 2**: For each carousel, scroll to it, click "Next" until disabled (extracting stores each step), then click "Previous" back to start — video shows the full right-then-left journey. Stores deduplicated by ID.

#### Proven extraction code (use this exact pattern):

```javascript
async (page) => {
  // === PHASE 1: Vertical scroll to discover all carousels ===
  const carouselPositions = new Map(); // carouselId -> { scrollY, title, secondPassScore }
  const scrollPositions = [0, 500, 1000, 1500, 2000, 2500, 3000, 3500, 4000, 4500, 5000];

  for (const scrollY of scrollPositions) {
    await page.evaluate((y) => window.scrollTo(0, y), scrollY);
    await page.waitForTimeout(800);

    const discovered = await page.evaluate((currentScrollY) => {
      const results = [];
      const allEls = document.querySelectorAll('div, span');
      for (const el of allEls) {
        const directText = Array.from(el.childNodes)
          .filter(n => n.nodeType === 3)
          .map(n => n.textContent.trim())
          .join(' ');
        if (!directText.match(/^carousel\.standard:store_carousel:\S+$/)) continue;
        const rect = el.getBoundingClientRect();
        if (rect.y < -800 || rect.y > 1500) continue;

        let container = el.parentElement;
        for (let i = 0; i < 3; i++) {
          if (!container) break;
          if (container.className?.includes('StyledStackChildren')) break;
          container = container.parentElement;
        }
        if (!container) continue;

        const containerText = container.innerText;
        const secondPassMatch = containerText.match(/Ranking Second Pass Score:\s*([\d.]+|N\/A)/);
        let title = 'Unknown';
        const lines = containerText.split('\n');
        for (let i = 0; i < lines.length; i++) {
          if (lines[i].includes('carousel.standard:store_carousel:') && i + 1 < lines.length) {
            const nextLine = lines[i + 1].trim();
            if (nextLine && !nextLine.startsWith('card.') && !nextLine.startsWith('Create') && nextLine.length < 80) {
              title = nextLine; break;
            }
          }
        }
        const absY = currentScrollY + rect.y;
        results.push({ carouselId: directText, title, secondPassScore: secondPassMatch ? secondPassMatch[1] : null, absY: Math.round(absY) });
      }
      return results;
    }, scrollY);

    for (const c of discovered) {
      const existing = carouselPositions.get(c.carouselId);
      if (!existing || (existing.title === 'Unknown' && c.title !== 'Unknown')) {
        carouselPositions.set(c.carouselId, c);
      }
    }
  }

  // === PHASE 2: Horizontal click-through for each carousel ===
  const allCarousels = [];

  for (const [carouselId, info] of carouselPositions) {
    const targetY = Math.max(0, info.absY - 200);
    await page.evaluate((y) => window.scrollTo(0, y), targetY);
    await page.waitForTimeout(600);

    const storeMap = new Map();

    const collectStores = async () => {
      const stores = await page.evaluate((cId) => {
        const allEls = document.querySelectorAll('div, span');
        for (const el of allEls) {
          const directText = Array.from(el.childNodes)
            .filter(n => n.nodeType === 3).map(n => n.textContent.trim()).join(' ');
          if (directText !== cId) continue;
          let container = el.parentElement;
          for (let i = 0; i < 3; i++) {
            if (!container) break;
            if (container.className?.includes('StyledStackChildren')) break;
            container = container.parentElement;
          }
          if (!container) return [];
          const containerText = container.innerText;
          const storeIds = [...new Set(containerText.match(/card\.store:store:\d+/g) || [])];
          const scores = containerText.match(/Store Ranking Score: [\d.]+/g) || [];
          const storeLinks = [...container.querySelectorAll('a[href*="/store/"]')];
          const storeNames = storeLinks.map(a => {
            const spans = a.querySelectorAll('span');
            for (const s of spans) {
              const t = s.textContent.trim();
              if (t.length > 2 && t.length < 50 && !t.includes('$') && !t.includes('min')
                  && !t.match(/^\d/) && !t.includes('Store Debug') && !t.includes('Ranking')
                  && !t.includes('Sponsored')) return t;
            }
            return null;
          }).filter(Boolean);
          return storeIds.map((id, i) => ({
            id: id.replace('card.store:store:', ''),
            name: storeNames[i] || null,
            score: scores[i] ? scores[i].replace('Store Ranking Score: ', '') : null
          }));
        }
        return [];
      }, carouselId);
      for (const s of stores) { if (!storeMap.has(s.id)) storeMap.set(s.id, s); }
    };

    await collectStores();

    // Click Next until disabled (scroll right through entire carousel)
    let clickCount = 0;
    const maxClicks = 20;
    while (clickCount < maxClicks) {
      const nextState = await page.evaluate((cId) => {
        const allEls = document.querySelectorAll('div, span');
        for (const el of allEls) {
          const directText = Array.from(el.childNodes)
            .filter(n => n.nodeType === 3).map(n => n.textContent.trim()).join(' ');
          if (directText !== cId) continue;
          let container = el.parentElement;
          for (let i = 0; i < 3; i++) {
            if (!container) break;
            if (container.className?.includes('StyledStackChildren')) break;
            container = container.parentElement;
          }
          if (!container) return 'no-container';
          const nextBtn = container.querySelector('[aria-label="Next button of carousel"]');
          if (!nextBtn) return 'no-button';
          if (nextBtn.disabled) return 'disabled';
          nextBtn.click();
          return 'clicked';
        }
        return 'no-carousel';
      }, carouselId);
      if (nextState !== 'clicked') break;
      clickCount++;
      await page.waitForTimeout(500);
      await collectStores();
    }

    // Click Previous back to start (video shows return journey)
    let prevCount = 0;
    while (prevCount < maxClicks) {
      const prevState = await page.evaluate((cId) => {
        const allEls = document.querySelectorAll('div, span');
        for (const el of allEls) {
          const directText = Array.from(el.childNodes)
            .filter(n => n.nodeType === 3).map(n => n.textContent.trim()).join(' ');
          if (directText !== cId) continue;
          let container = el.parentElement;
          for (let i = 0; i < 3; i++) {
            if (!container) break;
            if (container.className?.includes('StyledStackChildren')) break;
            container = container.parentElement;
          }
          if (!container) return 'no-container';
          const prevBtn = container.querySelector('[aria-label="Previous button of carousel"]');
          if (!prevBtn) return 'no-button';
          if (prevBtn.disabled) return 'disabled';
          prevBtn.click();
          return 'clicked';
        }
        return 'no-carousel';
      }, carouselId);
      if (prevState !== 'clicked') break;
      prevCount++;
      await page.waitForTimeout(500);
      await collectStores();
    }

    const stores = [...storeMap.values()];
    allCarousels.push({
      carouselId, title: info.title, secondPassScore: info.secondPassScore,
      storeCount: stores.length, nextClicks: clickCount, stores
    });
  }

  await page.evaluate(() => window.scrollTo(0, 0));
  return JSON.stringify(allCarousels, null, 2);
}
```

#### Key DOM facts (proven 2026-03-19):
- Debug overlay carousel IDs are **direct text nodes** in `<div>` elements — match via `el.childNodes` filtered to `nodeType === 3`
- Each carousel wrapper has class `StyledStackChildren-sc-*` — walk up max 3 parents to find it
- Carousel title is the **line immediately after** the carousel ID in `container.innerText`
- Store card IDs: regex `card\.store:store:\d+` in container text
- Store names: `<a href="/store/...">` links → first `<span>` child with length 3-50 and no `$`, `min`, `Store Debug`, or `Ranking` text
- Carousels lazy-load — must scroll in 500px increments with 800ms waits
- Carousels without `card.store:store:` matches use item/grocery cards (not store cards) — report as 0 stores
- `setTimeout` is NOT available in `browser_run_code` — use `page.waitForTimeout()` instead

**Horizontal scrolling (Phase 2):**
- "Next button of carousel" (`aria-label="Next button of carousel"`) — click to scroll right, disabled at end
- "Previous button of carousel" (`aria-label="Previous button of carousel"`) — click to scroll left, disabled at start
- Each click scrolls ~4 stores into view; wait 500ms for animation before extracting
- Carousels can have up to 20 stores — only ~4 visible at a time
- Deduplicate stores by ID across all horizontal positions
- Bottom bar Second Pass Score: regex `Ranking Second Pass Score:\s*([\d.]+|N\/A)` in container text

**Self-heal targets for carousel extraction**:
- If `StyledStackChildren` class name changes → walk up parents looking for any container that holds both carousel ID text and store card links
- If carousel ID format changes from `carousel.standard:store_carousel:` → look for any debug overlay text matching `carousel.*:store_carousel:` or similar patterns
- If `card.store:store:` format changes → look for any debug overlay text with numeric IDs following "store"
- If store name selector changes → fall back to link text content from `/store/` links
- If 0 carousels found → page structure may have changed fundamentally; take a snapshot (debug mode may be off at this point) and report what's visible

### 6. Console Errors

```
browser_console_messages → level: error
```

Record count and content of any JS errors. Known benign errors: React hydration #418/#425/#423 (SSR artifacts).

### 7. Close Browser and Capture Video

```
browser_close
```
- Finalizes session video — the `.webm` file only appears after this call completes
- Each session produces one file: `/tmp/playwright-mcp-output/<uuid>.webm`
- Find it: `VIDEO_FILE=$(ls -t /tmp/playwright-mcp-output/*.webm 2>/dev/null | head -1)`
- Copy to audit trail: `cp "${VIDEO_FILE}" <runDir>/session.webm`
- Create 2x speed version: `ffmpeg -i "<runDir>/session.webm" -filter:v "setpts=0.5*PTS" -an "<runDir>/session-2x.webm"`

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
| Pod not found / expired | Hard stop — sandbox needs recreation via `/sandbox-setup` |
| "Sign In" not found | Page may already be authenticated — check URL for `/home` |
| Login returns error | Re-check password field is filled on second step |
| "How Fees Work" not found | Skip — only appears on first visit |
| Carousel Next disabled | End of carousel — move to next one |
| Debug Mode toggle not found | Use fallback: checkbox near bottom-right. If that fails, search for "Debug" text |
| Debug overlays not appearing | Toggle may not have fired — verify checkbox `checked` state via evaluate |
| Page shows blank/spinner | Wait longer (10s), then retry navigate |
| Video not saved | Ensure `browser_close` is called — video writes on context close. Verify `~/.claude/playwright-mcp-config.json` has `recordVideo` and MCP server was restarted after config change |
| Video too large | Reduce size in `~/.claude/playwright-mcp-config.json` → `recordVideo.size` (e.g., `800x600`) |
| Latency measurement returns null | Use `performance.getEntriesByType('navigation')` fallback |
| Snapshot too large (token limit) | Debug mode is on — use `browser_evaluate`, never `browser_snapshot` |
| `setTimeout` not defined | In `browser_run_code`, use `page.waitForTimeout()` instead |
| Raw React JSON in extraction | Filter for direct text nodes only (`nodeType === 3`) |
| 0 carousels found | Page may have changed — disable debug mode, take snapshot, report structure |
| Selector class hash changed | Styled-component hashes change on builds — use fallback selectors |
| Click indicator not showing | Re-inject after any full page navigation; listener uses `capture: true` |
| ffmpeg not installed | Run `brew install ffmpeg`. If rejected, skip 2x video and note in report |

---

## Changelog

| Date | Change | Reason |
|---|---|---|
| 2026-03-19 | Initial playbook with proven patterns | First successful end-to-end test run |
| 2026-03-19 | Added self-healing protocol | UI will evolve; need automatic recovery + persistent fixes |
| 2026-03-19 | Added snapshot token limit warnings | Debug mode snapshots exceeded 700K chars |
| 2026-03-19 | Added `browser_run_code` extraction code | Generic "read snapshot" approach doesn't work with debug mode on |
| 2026-03-19 | Added pod liveness check as Step 0 | Without live pod, test runs against default feed, not sandbox code |
| 2026-03-19 | Fixed video recording — use `contextOptions.recordVideo` | `--save-video` is not a valid `@playwright/mcp` flag; config file approach works |
| 2026-03-19 | Added click indicator injection (Step 3b) | Neon green pulsing circles at click locations for video review |
| 2026-03-19 | Rewrote carousel extraction as two-phase (Step 5) | Phase 1: vertical scroll discovers carousels. Phase 2: horizontal Next/Previous click-through captures all stores (up to 20 per carousel) |
| 2026-03-19 | Added 2x speed video post-processing | `ffmpeg setpts=0.5*PTS` creates `session-2x.webm` for faster video review |
