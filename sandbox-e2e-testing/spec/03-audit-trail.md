# Audit Trail Contract

Reference spec for the per-run audit directory structure. Implemented in `sandbox-test.md`.

## Directory Structure

```
feed-service/.claude/sandbox/runs/<run-id>/
  meta.json                ← run metadata + per-load metrics
  steps.jsonl              ← append-only step log
  feed-service.log         ← raw kubectl pod logs
  load-{1,2,3}-carousels.json  ← per-load extraction data
  report.md                ← human-readable summary
  analyze.py               ← reproducible analysis script
  homepage-evidence.png    ← hero screenshot for PRs
  session.webm             ← browser session video (VP8, 1280x720)
  iguazu-validation.md     ← Snowflake validation (from validate-iguazu)
```

Run ID: `<timestamp>_<branch-sanitized>_<short-hash>`

## meta.json Schema

```json
{
  "runId": "...",
  "branch": "...",
  "commitHash": "...",
  "sandboxName": "...",
  "pod": "...",
  "startedAt": "ISO8601",
  "completedAt": "ISO8601",
  "status": "pass | fail | error",
  "loads": [
    { "loadNumber": 1, "latencyMs": 4733, "carouselCount": 34, "totalStores": 286, "requestId": "..." }
  ],
  "checks": {
    "avgHomepageLoadMs": 4242,
    "jsErrors": 0,
    "logErrors": 0,
    "logWarnings": 2,
    "loadsCompleted": 3,
    "loadsAttempted": 3
  }
}
```

## Pass/Fail Criteria

- **PASS**: All 3 loads completed, avg latency < 10s, avg carousels > 0, 0 JS errors
- **FAIL**: < 2 good loads after 5 attempts, avg latency >= 10s, any load returned 0 carousels, JS errors > 0

## steps.jsonl

One JSON per line, append-only: `{"ts":"ISO8601","step":"step-name","result":"description","ok":true}`

## Per-Load Files

`load-N-carousels.json` — full carousel/store extraction from a single homepage load. Ground truth for Snowflake validation.

## Retention

No pruning. Runs accumulate.
