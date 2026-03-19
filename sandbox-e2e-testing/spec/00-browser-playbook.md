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

### 6. Debug Mode (for ML score analysis)
```
click → "Debug Mode" button (bottom-right corner)
browser_take_screenshot → debug-mode.png
```
- Exposes ML ranking scores for comparison and analysis.

---

## Failure Recovery

| Failure | Recovery |
|---|---|
| "Sign In" not found | Page may already be authenticated — check URL for `/home` |
| Login returns error | Re-check password field is filled on second step |
| "How Fees Work" not found | Skip — only appears on first visit |
| Carousel Next disabled | Carousel may be at end — try a different carousel |
| Debug Mode not found | May need to scroll to bottom-right — evaluate page scroll position |
| Page shows blank/spinner | Wait longer (10s), then retry navigate |
