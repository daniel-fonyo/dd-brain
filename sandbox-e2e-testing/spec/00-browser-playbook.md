# Browser Interaction Playbook — doordashtest.com Homepage

Deterministic step sequence for sandbox-test browser interactions. Follow this known path by default. Only fall back to exploratory page analysis if a step fails or the UI has known breaking changes.

## Credentials

- **Domain**: `https://www.doordashtest.com/`
- **Email**: `tas-cx-doortest-egsn7vrtcl@doordash.com`
- **Password**: `1XQlCDr8Qb`
- **Note**: Test account on doortest tenant. Never use `doordash.com` — creds won't work there.

---

## Step Sequence

### 1. Navigate
```
browser_navigate → https://www.doordashtest.com/
```
- Landing page loads with "Sign In" / "Sign Up" buttons.

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

### 4. Screenshot homepage
```
browser_take_screenshot → homepage.png
```
- Verify: carousels visible ("Under $1 delivery fee", "Slam dunk savings", etc.)

### 5. Scroll carousel
```
click → "Next button of carousel" (first instance)
browser_take_screenshot → carousel-scrolled.png
```
- Verify: new stores appear, "Previous" button becomes enabled.

### 6. Enable Debug Mode (for ML score analysis)

The Debug Mode toggle is a checkbox inside a styled component, not a regular button.

```
evaluate → document.querySelector('.ToggleContainer-sc-ba2scp-0 input[type="checkbox"]').click()
  ↓ debug overlays appear on all page components
browser_take_screenshot → debug-mode-on.png
```

- **Toggle selector**: `.ToggleContainer-sc-ba2scp-0 input[type="checkbox"]` (bottom-right corner)
- **Fallback**: If class name changes, look for a checkbox near "Debug Mode" text at bottom-right of page
- Clicking again disables debug overlays

### 7. Analyze Debug Overlays

Once Debug Mode is ON, the page shows:

**Page-level overlay (top):**
- `PAGE LEVEL LOG` header
- Component IDs for every element: banners, carousels, cards, containers

**Per-carousel labels:**
- Format: `carousel.standard:store_carousel:<carousel_id>`
- Examples: `under_1_delivery`, `slam_dunk_savings`, `eat`, `fastest_near_you`

**Per-store-card overlays:**
- Format: `card.store:store:<store_id>` (e.g., `card.store:store:297366`)
- **Store Ranking Score**: float (e.g., `0.0006`, `0.0005`) — per-card ML ranking score
- **Store Debug Tools** button on each card

**Bottom bar (persistent):**
- **Ranking First Pass Score**: usually `N/A` on homepage (first pass is search-specific)
- **Ranking Second Pass Score**: float (e.g., `0.0060`, `0.0265`) — aggregate page-level re-ranking score
- **ML Debugging Information** link

```
browser_take_screenshot → debug-store-cards.png   (capture ranking scores)
scroll down → capture more carousels
browser_take_screenshot → debug-more-carousels.png
```

### 8. Store Debug Tools (per-store deep dive)

Click "Store Debug Tools" on any store card to open a modal with:

```
click → "Store Debug Tools" button on a store card
  ↓ modal opens: "<Store Name>-<store_id>"
```

| Tool | Purpose |
|---|---|
| MX Tools | Merchant Details Page |
| MX Portal | Merchant Portal Page |
| Search Diagnostic Tools | Debug why store doesn't appear in search results |
| Search Evaluation Tools | Debug store ranking |

```
click → "Close" button to dismiss modal
```

---

## Debug Mode — ML Score Reference

| Score | Location | Meaning |
|---|---|---|
| Store Ranking Score | Per store card | Individual store relevance score from ranking model |
| Ranking First Pass Score | Bottom bar | Initial candidate retrieval score (N/A on homepage, used in search) |
| Ranking Second Pass Score | Bottom bar | Re-ranking score after applying all ranking features |

**Observed score ranges (sandbox, consumer 757606047):**
- Store Ranking Scores: `0.0004` – `0.0006` (narrow range, similar relevance)
- Ranking Second Pass Score: `0.0060` (Under $1 delivery carousel), `0.0265` (Get breakfast carousel)
- Ranking First Pass Score: `N/A` on all homepage carousels

---

## Failure Recovery

| Failure | Recovery |
|---|---|
| "Sign In" not found | Page may already be authenticated — check URL for `/home` |
| Login returns error | Re-check password field is filled on second step |
| "How Fees Work" not found | Skip — only appears on first visit |
| Carousel Next disabled | Carousel may be at end — try a different carousel |
| Debug Mode toggle not found | Class name may have changed — search for `input[type="checkbox"]` near bottom-right |
| Debug overlays not appearing | Toggle may not have fired — verify checkbox `checked` state via evaluate |
| Store Debug Tools modal stuck | Click "Close" button or press Escape |
| Page shows blank/spinner | Wait longer (10s), then retry navigate |
