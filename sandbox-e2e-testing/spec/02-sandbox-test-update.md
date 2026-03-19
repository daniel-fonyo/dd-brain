# Spec 02 — sandbox-test updates

## Goal
Extend `sandbox-test` to write into a caller-supplied `runDir` instead of `/tmp`. Return structured result JSON for `sandbox-run` to consume.

---

## Changes

### Accept runDir

At Step 1 (Load Sandbox State), also check for a `runDir` parameter passed by the caller.

- If `runDir` is set → write all outputs there
- If not set (standalone invocation) → fall back to `/tmp/sandbox-<timestamp>/` and create it

### Log capture destination

Change:
```bash
# old
kubectl ... | tee /tmp/sandbox-logs.txt

# new
kubectl ... | tee "${RUN_DIR}/feed-service.log"
```

### Screenshot destination

Save all Playwright screenshots to `${RUN_DIR}/screenshots/` with zero-padded sequence names:
- `01-homepage.png`
- `02-carousel-<sanitized-title>.png` (one per carousel)

### steps.jsonl integration

At each step, append a line to `${RUN_DIR}/steps.jsonl`:
```bash
echo '{"ts":"<now>","step":"<name>","result":"<desc>"}' >> "${RUN_DIR}/steps.jsonl"
```

Steps to append:
- `log-capture-start`
- `browser-activate-routing`
- `browser-navigate`
- `browser-screenshot` (each)
- `browser-carousels` (with count + titles)
- `browser-scroll`
- `browser-console-errors` (count)
- `log-capture-stop` (with line count, error count, warning count)

### Structured result output

At Step 5 (Report Results), before printing the table, write a result JSON to `${RUN_DIR}/result.json`:

```json
{
  "homepageLoaded": true,
  "carousels": ["Featured Stores", "Items for You", "Deals", "Trending"],
  "jsErrors": 0,
  "logLines": 452,
  "logErrors": 0,
  "logWarnings": 2,
  "screenshots": ["01-homepage.png", "02-carousel-featured-stores.png"],
  "status": "pass"
}
```

`sandbox-run` reads this file to populate `meta.json` checks and `report.md`.

---

## Backward Compatibility

When invoked standalone (no runDir), behavior is identical to current — `/tmp` fallback, no steps.jsonl, no result.json required. The inline report table is always printed.
