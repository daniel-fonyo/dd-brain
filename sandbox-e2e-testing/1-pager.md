# Feed Service Homepage E2E Sandbox Testing — 1-Pager

## Problem

Validating homepage ranking changes today is manual and slow. Engineers spin up sandbox environments, load the homepage, eyeball the feed, check logs, and try to correlate events and experiment assignments by hand. This is error-prone, doesn't scale, and creates risk when shipping ranking changes.

## Goal

Automate end-to-end homepage validation in sandbox environments so engineers get fast, reproducible signal that:
- The feed loaded correctly
- The right DVs/experiments were applied
- Snowflake events were emitted as expected
- Visual ranking and surface appearance is correct

## Scope

### In Scope
- Spinning up sandbox environments on demand
- Reloading the homepage and triggering a full feed load cycle
- Asserting Snowflake events emitted on page load (impression events, ranking signals)
- Asserting DVs and experiment assignments are present and correct
- Visual inspection of homepage feed (ranking order, surface appearance)
- Basic interactions: scroll, tap/click cards, carousel navigation
- Log analysis: surface errors, unexpected ranking decisions, blending anomalies

### Out of Scope (for now)
- Multi-user / multi-session load testing
- Full regression coverage beyond homepage
- Non-sandbox environments (staging, prod)

## Approach

### 1. Sandbox Provisioning
- Reuse existing sandbox-setup skill/tooling to spin up or resync sandbox env
- Parameterize by user profile, DV overrides, experiment config

### 2. Homepage Load + Log Capture
- Automate homepage navigation (Playwright)
- Capture all network requests, console logs, server-side logs from feed-service
- Assert no load errors, expected response shape from feed ranking endpoint

### 3. Snowflake Event Validation
- After load, query Snowflake (or event sink) for expected impression/engagement events
- Assert event presence, correct item IDs, expected ranking positions

### 4. DV / Experiment Assertion
- Parse feed response or server logs for DV assignment payloads
- Assert expected experiments are enrolled, correct variant applied

### 5. Visual Inspection
- Screenshot homepage at load
- Playwright snapshot for accessibility/structural validation
- Optionally: image diff against baseline to catch visual regressions

### 6. Ranking Sanity Checks
- Assert top-N items match expected ranking heuristics (e.g. no obviously wrong content surfaced)
- Flag anomalies: duplicate content, wrong content type, ranking inversions

## Output / Reporting
- Pass/fail summary per assertion category
- Linked logs, screenshots, and Snowflake event traces for failures
- CI-compatible: can be triggered pre-merge or on a schedule

## Success Criteria
- An engineer can run a single command and get a green/red signal on homepage health
- False positive rate is low enough that failures are actionable
- Covers the most common ranking regression patterns caught manually today

## Open Questions
- What's the right event sink to query — Snowflake directly, or an intermediate event store?
- How do we parameterize DV/experiment overrides in sandbox?
- Should visual diffs be opt-in or always-on?
- What latency is acceptable for the full test run?

## Next Steps
1. Audit existing sandbox-setup and sandbox-test skills for reuse
2. Define the minimal set of Snowflake events to assert
3. Prototype Playwright-based homepage load + log capture
4. Define DV/experiment assertion contract with feed-service response schema
