# Intent Prediction Score Logging - E2E Test Report

**Date**: 2026-03-31 02:47-02:49 UTC
**Branch**: `feat/intent-prediction-score-logging` (commit `e302010`)
**Pod**: `feed-service-web-group1-sandbox-rd4ae1c9-ffb4bc698-k7z7z`
**Namespace**: `feed-service-sandbox`

## Test Results Summary

| Check | Result | Notes |
|-------|--------|-------|
| Pod running | PASS | Pod phase: Running, warmup completed in 88,969ms |
| Service healthy | PASS | Warmup requests processed successfully, Sibyl predictions returning |
| Browser login | PASS | Signed in via identity.doordash.com, redirected to /home |
| Homepage rendered | PASS | Carousels visible (Most loved, Fastest near you), address = 1111 Brickell Ave |
| Sandbox routing active | FAIL | No homepage gRPC requests reached the sandbox pod during browser session |
| `vertical_intent_prediction_score` in logs | INCONCLUSIVE | Cannot verify -- no requests reached the pod |
| `score_modifier` in logs | INCONCLUSIVE | Same reason |
| Iguazu events fired | INCONCLUSIVE | ENABLE_SANDBOX_IGUAZU not found (expected -- IguazuModule.kt compile may have failed) |

**Overall: INCONCLUSIVE** -- the feature code was never exercised because the sandbox pod did not receive browser traffic.

## Root Cause Analysis

### Why no traffic reached the sandbox pod

1. **Code sync source mismatch**: The main `feed-service` checkout is on `master` branch, NOT `feat/intent-prediction-score-logging`. The worktree at `.claude/worktrees/intent-prediction-score-logging/` has the correct code, but `devbox run web-group1-remote` syncs from the main checkout directory (`/Users/daniel.fonyo/Projects/feed-service/`).

2. **Implication**: Even if sandbox routing was working, the pod would be running `master` code, not the feature branch. The 13-file diff (8 prod + 4 test + 1 IguazuModule bypass) was never deployed to the pod.

3. **Sandbox routing uncertainty**: The routing URL (`/developer/sandbox/...`) was visited and redirected to `/home`. However, no gRPC requests appeared in the pod logs during the browser session (02:47-02:49 UTC). All homepage-related logs were from warmup (~02:45-02:46). This could mean:
   - The routing cookie expired or wasn't properly set
   - The frontend fetched the feed from a CDN cache instead of making a fresh backend call
   - The sandbox routing mechanism only intercepts certain API paths

### IguazuModule.kt compile issue

The `:platform:compileKotlin` failure mentioned in the task description means the `val isSandboxEnv = false` change in `IguazuModule.kt` may not have been compiled into the running JAR. The logs confirm this -- `ENABLE_SANDBOX_IGUAZU` environment variable lookup is happening (the default code path), meaning the hardcoded bypass is NOT active. Even if events were generated, they would be blocked by sandbox Iguazu gating.

## What Worked

- **Pod is healthy**: The sandbox pod started, warmed up, and processes requests (warmup consumer_id 90520403 and 255790425 both worked)
- **Browser automation**: Playwright MCP tools worked smoothly -- navigated, signed in, confirmed homepage load with screenshot
- **kubectl access**: All log queries executed without permission issues
- **Login flow**: Two-step identity flow (email -> password) completed successfully

## What Didn't Work

- **No live traffic**: The sandbox pod received zero homepage requests from the browser session
- **Code not deployed**: Feature branch code was in a worktree, not on the main checkout that `devbox run web-group1-remote` syncs from
- **Iguazu bypass ineffective**: The `:platform` compile error means the bypass never took effect

## Observations About the Autonomous Workflow

### Strengths
- kubectl log analysis is fast and effective for diagnosing pod state
- Playwright browser automation is reliable for the login + homepage flow
- The snapshot/screenshot approach gives good confidence the page loaded correctly

### Weaknesses
1. **No way to verify code sync**: There's no log or indicator showing WHICH code version is running on the pod. A startup log with git commit hash would be invaluable.
2. **Sandbox routing is a black box**: No feedback on whether the routing cookie was set or if requests are actually being intercepted.
3. **Worktree vs main checkout confusion**: The `devbox run web-group1-remote` sync tool syncs from the main checkout, but our changes are in a worktree. This is a systematic workflow gap.

### Timing
- Pod log check: ~5 seconds
- Browser sign-in flow: ~30 seconds
- Homepage load + screenshot: ~10 seconds
- Log analysis after browsing: ~15 seconds
- Total active test time: ~2 minutes

## Recommendations

### Immediate (to complete this test)
1. **Sync from worktree**: Run `devbox run web-group1-remote` from inside the worktree directory (`.claude/worktrees/intent-prediction-score-logging/`), or checkout the feature branch on the main feed-service directory before syncing.
2. **Re-activate routing**: Visit the sandbox routing URL again after sync, then browse the homepage.
3. **Fix IguazuModule compile**: If Snowflake validation is needed, ensure the `:platform` module compiles cleanly with the `val isSandboxEnv = false` change.

### Process Improvements
1. **Add git commit hash to startup logs**: This would make it instantly clear whether the synced code matches expectations.
2. **Add a sandbox routing verification endpoint**: A simple `/debug/routing` endpoint that returns the pod name would confirm traffic is hitting the right pod.
3. **Document worktree sync workflow**: CLAUDE.md should note that `devbox run web-group1-remote` must be run from the directory containing the code to sync, not necessarily the repo root.
4. **Add DEBUG logging for score_modifiers**: Temporarily add a log line that prints the full `score_modifiers` map before Iguazu event creation, so we can see the data even if Iguazu is blocked.

### For Future Autonomous E2E Loops
1. Always verify code sync BEFORE browser testing (check pod startup logs or a version endpoint)
2. After visiting the routing URL, make a curl request to the pod's gRPC endpoint directly to confirm routing
3. Consider using `kubectl port-forward` + direct gRPC call as an alternative to browser-based testing
