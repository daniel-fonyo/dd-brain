# Spec 02 — sandbox-test updates

## Goal
Extend `sandbox-test` to write into a caller-supplied `runDir` instead of `/tmp`. Return structured result JSON for `sandbox-run` to consume. Output video instead of screenshots. Gather full carousel + store ML score inventory.

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

### Video (replaces screenshots)

Session video is recorded automatically by Playwright MCP (`--save-video=1280x720`). After `browser_close`:
```bash
cp /tmp/playwright-mcp-output/*.webm "${RUN_DIR}/session.webm"
```

No individual screenshots are taken.

### steps.jsonl integration

At each step, append a line to `${RUN_DIR}/steps.jsonl`:
```bash
echo '{"ts":"<now>","step":"<name>","result":"<desc>"}' >> "${RUN_DIR}/steps.jsonl"
```

Steps to append:
- `log-capture-start`
- `browser-activate-routing`
- `browser-navigate` (with `homepageLoadMs`)
- `browser-sign-in` (if needed)
- `browser-debug-mode-on`
- `browser-carousel` (one per carousel, with name, ID, store count)
- `browser-console-errors` (count)
- `browser-close` (video finalized)
- `log-capture-stop` (with line count, error count, warning count)

### Structured result output

At Step 12 (Report Results), before printing the report, write a result JSON to `${RUN_DIR}/result.json`:

```json
{
  "homepageLoadMs": 2340,
  "carouselCount": 4,
  "totalStoresObserved": 28,
  "carousels": [
    {
      "name": "Under $1 delivery fee",
      "carouselId": "carousel.standard:store_carousel:under_1_delivery",
      "rankingSecondPassScore": 0.0060,
      "stores": [
        {
          "name": "Lucca Delicatessen",
          "storeId": "297366",
          "rankingScore": 0.0006,
          "rating": 4.8,
          "deliveryFee": "$0.99",
          "deliveryTime": "25 min"
        }
      ]
    }
  ],
  "jsErrors": 0,
  "logLines": 452,
  "logErrors": 0,
  "logWarnings": 2,
  "video": "session.webm",
  "status": "pass"
}
```

`sandbox-run` reads this file to populate `meta.json` checks and `report.md`.

---

## Backward Compatibility

When invoked standalone (no runDir), behavior is identical to current — `/tmp` fallback, no steps.jsonl, no result.json required. The inline report (checks table + carousel inventory + score analysis) is always printed.
