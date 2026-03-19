# Spec 01 — sandbox-run skill

## File
`~/.claude/commands/sandbox-run.md`

## Description
Top-level orchestrator. Entry point for "test this change on homepage." Figures out sandbox state, syncs changes, runs test, captures audit trail.

---

## Steps

### 1. Init Audit Run

Generate a run ID:
```bash
RUN_ID="$(date -u +%Y-%m-%dT%H-%M-%S)-$(openssl rand -hex 3)"
RUN_DIR="/Users/daniel.fonyo/Projects/feed-service/.claude/sandbox/runs/${RUN_ID}"
mkdir -p "${RUN_DIR}/screenshots"
```

Write initial `meta.json` (partial — completedAt/status/checks filled at end).
Append first step to `steps.jsonl`:
```jsonl
{"ts":"<now>","step":"run-start","result":"run <runId> initialized"}
```

Report: `"Starting sandbox run <runId>"`

---

### 2. Read State

Read `~/.claude/sandbox-state.json`. Extract `sandboxName`, `pod`, `homepageUrl`.

Append to steps.jsonl.

If missing → append `{"step":"state-read","result":"no state found","ok":false}` → invoke `/sandbox-setup`, re-read state after.

---

### 3. Verify Branch

```bash
cd /Users/daniel.fonyo/Projects/feed-service && git branch --show-current
```

If user supplied a branch → check it out first.
If no branch supplied → confirm current branch with user if not `master`.

Append to steps.jsonl: `{"step":"branch-verify","result":"<branch>"}`

Report: `"feed-service branch: <branch>"`

---

### 4. Pod Liveness Check

```bash
kubectl get pod <pod> -n feed-service-sandbox --no-headers 2>&1
```

- **Running** → proceed to Step 5
- **Not found / Error / Not Running** → append `{"step":"pod-check","result":"pod not alive","ok":false}` → invoke `/sandbox-setup` → re-read state → proceed to Step 5

Append to steps.jsonl: `{"step":"pod-check","result":"Running | <error>"}`

---

### 5. Sync Decision

**Skip this step entirely if running in CI** — CI always syncs on commit via its own pipeline.

Check whether a sync is needed:

```bash
cd /Users/daniel.fonyo/Projects/feed-service && git status --porcelain
```

Sync if **either**:
- User request contains "resync", "retest", "sync", or "latest changes"
- `git status --porcelain` has any output (local changes present)

If no sync needed → append `{"step":"devbox-run","result":"skipped — no local changes"}` → proceed to Step 6.

If syncing → from `/Users/daniel.fonyo/Projects/feed-service`, run:
```bash
devbox run web-group1-remote
```
Wait for `======== Service is ready now ========`. Update `~/.claude/sandbox-state.json` and `meta.json` if pod/url changed.

Append: `{"step":"devbox-run-ready","result":"service ready | skipped"}`

Report: `"Sandbox synced and ready"` or `"Sandbox already up-to-date — skipping sync"`

---

### 6. Run Homepage Test

Pass `RUN_DIR` to sandbox-test. Sandbox-test will:
- Write pod logs to `${RUN_DIR}/feed-service.log`
- Write screenshots to `${RUN_DIR}/screenshots/`
- Append steps to `${RUN_DIR}/steps.jsonl`
- Return structured result JSON

Store the result for Step 7.

---

### 7. Finalize Audit

Write completed `meta.json` with `completedAt`, `status`, `checks` from test result.

Write `report.md` using template from `spec/03-audit-trail.md`.

Update latest symlink:
```bash
ln -sfn "${RUN_DIR}" "/Users/daniel.fonyo/Projects/feed-service/.claude/sandbox/latest"
```

Append: `{"step":"run-complete","result":"<status>","ok":<bool>}`

---

### 8. Report to User

Print inline report (table format from report.md). Include path to full run:
```
Full run: feed-service/.claude/sandbox/runs/<runId>/
```
