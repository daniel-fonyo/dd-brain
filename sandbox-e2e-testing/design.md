# Sandbox E2E — Architecture Design

## Status
Designing next steps from current skill baseline. Implementing the minimal agentic layer to support "test this change on homepage."

## Current State
- `sandbox-setup.md` — provisions sandbox, constructs state, verifies branch
- `sandbox-test.md` — homepage browser interaction + kubectl log capture
- `sandbox-state.json` — persisted sandbox metadata (`~/.claude/`)
- No orchestrator. No audit trail. User must know to run setup first.

---

## Goal

User says: `"test this change on homepage"`

System should:
1. Figure out if sandbox is already running
2. Sync the right branch + local changes
3. Interact with homepage
4. Capture logs and debug output
5. Report findings inline
6. Persist all run data as an audit trail

---

## Architecture

### Three-skill model

```
sandbox-run        ← top-level orchestrator (NEW)
├── sandbox-setup  ← invoked if sandbox isn't alive (existing, unchanged)
└── sandbox-test   ← homepage interaction + log capture (existing, extended)
```

Each skill remains independently invocable. `sandbox-run` composes them.

---

### New skill: `sandbox-run`

Entry point for "test this change". Smart about what to do based on current state.

**Flow:**

```
1. Read sandbox-state.json
   └── if missing → invoke sandbox-setup

2. Verify branch
   └── user supplied branch? → checkout
   └── else → confirm current branch

3. Readiness check
   kubectl get pod <pod> -n feed-service-sandbox
   └── Running → skip setup, proceed
   └── Missing/Error → invoke sandbox-setup first

4. Sync changes
   devbox run web-group1-remote (background)
   └── wait for "Service is ready now"
   └── update state if pod/url changed

5. Init audit run
   create feed-service/.claude/sandbox/runs/<run-id>/
   write meta.json + open steps.jsonl

6. Invoke sandbox-test
   └── passes run dir so test writes logs/screenshots there

7. Finalize audit
   write report.md
   update runs/latest symlink

8. Report to user (inline summary + path to full run)
```

---

### Audit Trail

**Location:** `feed-service/.claude/sandbox/`

```
feed-service/.claude/sandbox/
  runs/
    2026-03-19T14-30-00-a1b2c3/
      meta.json          ← run metadata (branch, sandbox, status, check results)
      steps.jsonl        ← append-only step log (one JSON per line)
      feed-service.log   ← raw kubectl pod logs
      screenshots/
        01-homepage.png
        02-carousel-stores.png
        ...
      report.md          ← human-readable summary
  latest/                ← symlink → most recent run dir
```

**meta.json:**
```json
{
  "runId": "2026-03-19T14-30-00-a1b2c3",
  "trigger": "user: test this change on homepage",
  "branch": "feat/dfonyo-vertical-ranking-debug-logs",
  "sandboxName": "rd29ad35",
  "pod": "feed-service-web-group1-sandbox-rd29ad35-57b48c758d-pfpgc",
  "startedAt": "2026-03-19T14:30:00Z",
  "completedAt": "2026-03-19T14:35:12Z",
  "status": "pass",
  "checks": {
    "homepageLoaded": true,
    "carouselsFound": 4,
    "jsErrors": 0,
    "logErrors": 0,
    "logWarnings": 2
  }
}
```

**steps.jsonl** (append one line per step as it happens):
```jsonl
{"ts":"...","step":"state-read","result":"sandbox rd29ad35 found"}
{"ts":"...","step":"pod-check","result":"Running"}
{"ts":"...","step":"branch-verify","result":"feat/dfonyo-vertical-ranking-debug-logs"}
{"ts":"...","step":"devbox-run","result":"service ready at 14:30:45"}
{"ts":"...","step":"log-capture-start","result":"kubectl streaming to feed-service.log"}
{"ts":"...","step":"browser-navigate","result":"homepage loaded, 4 carousels found"}
{"ts":"...","step":"log-capture-stop","result":"452 lines captured, 0 errors, 2 warnings"}
{"ts":"...","step":"report","result":"pass"}
```

---

### Updates to `sandbox-test`

Extend to accept a `runDir` parameter (path to current audit run directory):
- Write pod logs to `<runDir>/feed-service.log` instead of `/tmp/sandbox-logs.txt`
- Save screenshots to `<runDir>/screenshots/`
- Append steps to `<runDir>/steps.jsonl`
- Return structured result JSON for `sandbox-run` to write into `meta.json`

---

## Claude Agentic Best Practices Applied

| Practice | Implementation |
|---|---|
| Persistent shared state | `sandbox-state.json` read/written across skill invocations |
| Audit-first | Append to `steps.jsonl` at every step, not just at the end |
| Composable skills | Each skill works standalone; sandbox-run composes them |
| Background tasks | kubectl log stream runs in background while browser interacts |
| Fail fast | Pod liveness check before attempting test |
| Evidence capture | Screenshots + raw logs stored per-run, never overwritten |
| Human-readable output | `report.md` alongside structured `meta.json` |
| Natural language trigger | "test this change" → sandbox-run figures out what's needed |

---

## Implementation Order

1. **`sandbox-run.md`** — new orchestrator skill
   - Readiness check (pod liveness)
   - devbox run gating
   - Audit init + finalize
   - Delegates to existing setup/test

2. **`sandbox-test.md` update** — accept `runDir`, write there instead of `/tmp`

3. **Audit directory scaffold** — ensure `feed-service/.claude/sandbox/runs/` is gitignored

4. **`report.md` template** — standardized summary format written at run end

---

## Open Questions
- Should `devbox run` stay alive between test runs, or always re-sync?
- How many runs to retain before pruning old entries?
- Should `steps.jsonl` be readable by future Claude sessions for pattern analysis?
