# Spec 03 — Audit Trail

## Directory Structure

```
feed-service/.claude/sandbox/
  runs/
    <run-id>/
      meta.json
      steps.jsonl
      feed-service.log
      screenshots/
        01-homepage.png
        02-carousel-<title>.png
        ...
      report.md
  latest -> runs/<most-recent-run-id>   (symlink)
```

`<run-id>` format: `YYYY-MM-DDTHH-MM-SS-<6-char hex>` e.g. `2026-03-19T14-30-00-a1b2c3`

Generate hex suffix: `openssl rand -hex 3`

---

## meta.json

Written by `sandbox-run` at run start (partial), updated at run end (complete).

```json
{
  "runId": "2026-03-19T14-30-00-a1b2c3",
  "trigger": "<verbatim user request>",
  "branch": "feat/dfonyo-vertical-ranking-debug-logs",
  "sandboxName": "rd29ad35",
  "pod": "feed-service-web-group1-sandbox-rd29ad35-57b48c758d-pfpgc",
  "startedAt": "2026-03-19T14:30:00Z",
  "completedAt": "2026-03-19T14:35:12Z",
  "status": "pass | fail | error",
  "checks": {
    "homepageLoaded": true,
    "carouselsFound": 4,
    "jsErrors": 0,
    "logErrors": 0,
    "logWarnings": 2
  }
}
```

Fields written at start: `runId`, `trigger`, `branch`, `sandboxName`, `pod`, `startedAt`
Fields written at end: `completedAt`, `status`, `checks`

---

## steps.jsonl

**Purpose:** Claude's structured run memory. When a user asks "what happened last time?" or "why did the last test fail?", Claude reads `steps.jsonl` from the latest run dir to reconstruct the sequence of events without re-running anything. Also enables cross-run queries like "which step keeps failing?" by scanning multiple run dirs. Keep this file — it is the primary mechanism for Claude to reason about past runs.

Append-only. One JSON object per line. Written by any skill participating in the run.

Schema per line:
```json
{"ts": "<ISO8601>", "step": "<step-name>", "result": "<short description>", "ok": true}
```

`ok` is optional — omit for informational steps, include `false` for failures.

Example:
```jsonl
{"ts":"2026-03-19T14:30:00Z","step":"state-read","result":"sandbox rd29ad35 found"}
{"ts":"2026-03-19T14:30:01Z","step":"pod-check","result":"Running"}
{"ts":"2026-03-19T14:30:02Z","step":"branch-verify","result":"feat/dfonyo-vertical-ranking-debug-logs"}
{"ts":"2026-03-19T14:30:05Z","step":"devbox-run-start","result":"started"}
{"ts":"2026-03-19T14:30:45Z","step":"devbox-run-ready","result":"service ready"}
{"ts":"2026-03-19T14:30:46Z","step":"log-capture-start","result":"kubectl streaming"}
{"ts":"2026-03-19T14:30:47Z","step":"browser-activate-routing","result":"sandbox cookies set"}
{"ts":"2026-03-19T14:30:50Z","step":"browser-navigate","result":"homepage loaded"}
{"ts":"2026-03-19T14:30:52Z","step":"browser-screenshot","result":"01-homepage.png"}
{"ts":"2026-03-19T14:31:10Z","step":"browser-carousels","result":"4 found: Featured, Stores, Items, Deals"}
{"ts":"2026-03-19T14:31:45Z","step":"browser-scroll","result":"full page scrolled"}
{"ts":"2026-03-19T14:31:46Z","step":"log-capture-stop","result":"452 lines, 0 errors, 2 warnings"}
{"ts":"2026-03-19T14:31:47Z","step":"report","result":"pass","ok":true}
```

---

## feed-service.log

Raw kubectl pod log output captured during the browser session (t_start → t_end window).

Written by: `sandbox-test` (piped from background kubectl process).

---

## report.md

Written by `sandbox-run` at the end of the run. Human-readable summary.

Template:
```markdown
# Sandbox Run Report
**Run:** <runId>
**Branch:** <branch>
**Sandbox:** <sandboxName>
**Time:** <startedAt> → <completedAt>

## Results

| Check | Status | Notes |
|---|---|---|
| Homepage loaded | ✅ Pass | |
| Carousels found | ✅ Pass | 4: Featured, Stores, Items, Deals |
| JS errors | ✅ Pass | 0 |
| Log errors | ✅ Pass | 0 errors, 2 warnings |

**Overall: PASS**

## Warnings
- [list any WARN lines from logs]

## Failures
- [list any failures]

## Screenshots
- 01-homepage.png
- 02-carousel-featured.png
- ...
```

---

## Gitignore

Add to `feed-service/.gitignore`:
```
.claude/
```

---

## Retention

No pruning policy yet. Runs accumulate. Keep all for now — address in Phase 2.
