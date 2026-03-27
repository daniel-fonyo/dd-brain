# AFM Unpin Exemption — Test Results

**PR**: [#62270](https://github.com/doordash/feed-service/pull/62270)
**Test Plan**: [test-plan.md](../test-plan.md)
**Started**: 2026-03-26
**Status**: IN PROGRESS

---

## Environment

| Field | Value |
|-------|-------|
| Sandbox URL | `https://www.doordashtest.com/` |
| Address | 1111 Brickell Ave, Miami |
| Consumer ID | `757606047L` (hardcoded) |
| PR branch | `pr-62270` |
| feed-service worktree | `.claude/worktrees/afm-unpin-test` |

---

## Setup Log

| Step | Status | Notes |
|------|--------|-------|
| feed-service worktree created | PENDING | |
| Consumer ID hardcoded | PENDING | |
| Debug instrumentation added | PENDING | |
| Sandbox synced (`devbox run web-group1-remote`) | PENDING | |
| `[AFM_DEBUG]` lines confirmed in pod logs | PENDING | |

---

## Carousel ID Mapping

> Established during T1, reused across all tests.

| Carousel ID | Display Name | Type |
|-------------|-------------|------|
| *TBD* | Shop the Meal Box | Item |
| *TBD* | Best $12 meals in Miami | Store |

---

## Test 1: Baseline — DV OFF

**Executed**: *pending*
**DV `enable_afm_unpin_exemption`**: `false`

### Debug Log Output
```
<!-- Raw [AFM_DEBUG] output -->
```

### Carousel Position Map
| Position | Carousel ID | Display Name | Boost Status |
|----------|-------------|-------------|--------------|
| | | | |

### Screenshots
<!-- Link to results/t1-homepage-full.png -->

### Scroll Recording
<!-- Link to results/t1-homepage-scroll.mp4 -->

### Pass Criteria
- [ ] Debug logs show `DV=false`
- [ ] AFM carousels show `isAfmCarousel=false` and `eligible=true`
- [ ] AFM carousel IDs appear in `unboosted_order`
- [ ] Screenshot captured

### Analysis
<!-- Observations, unexpected behavior, notes -->

---

## Test 2: DV ON — AFM Pinned

**Executed**: *pending*
**DV `enable_afm_unpin_exemption`**: `true`

### Debug Log Output
```
<!-- Raw [AFM_DEBUG] output -->
```

### AFM Detection Detail
```
<!-- isAfmCarousel log lines showing campaign tags -->
```

### Carousel Position Map
| Position | Carousel ID | Display Name | Boost Status |
|----------|-------------|-------------|--------------|
| | | | |

### Position Delta — T1 vs T2
| Carousel | T1 Position | T2 Position | Delta | Expected? |
|----------|------------|------------|-------|-----------|
| Shop the Meal Box (item) | | | | |
| Best $12 meals in Miami (store) | | | | |

### Screenshots
<!-- Link to results/t2-homepage-full.png -->

### Scroll Recording
<!-- Link to results/t2-homepage-scroll.mp4 -->

### T1 vs T2 Side-by-Side
<!-- Link to results/t1-vs-t2-comparison.png -->

### Pass Criteria
- [ ] Debug logs show `DV=true`
- [ ] AFM carousels show `isAfmCarousel=true` and `eligible=false`
- [ ] `hasAfmVertical=true` + `hasAfmCampaign=true` confirmed
- [ ] AFM carousel IDs in `boosted_order`
- [ ] "Shop the Meal Box" near top of homepage
- [ ] "Best $12 meals in Miami" at pinned position
- [ ] Visual position shift from T1
- [ ] Other NV carousels still `eligible=true`

### Analysis
<!-- Campaign tags observed, position shift magnitude, surprises -->

---

## Test 3: Regression — Non-AFM NV Still Unpinned

**Executed**: *pending*
**DV `enable_afm_unpin_exemption`**: `true` (same as T2)

### Debug Log Output (non-AFM only)
```
<!-- Filtered [AFM_DEBUG] lines for non-AFM NV carousels -->
```

### Boosted/Unboosted Lists
```
<!-- Log Point 4 output -->
```

### Non-AFM NV Carousel Inventory
| Carousel ID | Vertical IDs | Display Name | `eligible` | In `unboosted_order`? |
|-------------|-------------|-------------|------------|----------------------|
| | | | | |

### Pass Criteria
- [ ] At least one non-AFM NV carousel found
- [ ] All non-AFM NV carousels show `eligible=true`
- [ ] Non-AFM NV carousel IDs in `unboosted_order`
- [ ] Restaurant carousels not in eligibility logs

### Analysis
<!-- Which NV verticals present, any unexpected behavior -->

---

## Final Rubric

| # | Assertion | Result |
|---|-----------|--------|
| 1 | T1: `DV=false` and `isAfmCarousel=false` confirmed | |
| 2 | T1: AFM carousels `eligible=true`, in `unboosted_order` | |
| 3 | T2: `DV=true` and `isAfmCarousel=true` confirmed | |
| 4 | T2: AFM carousels `eligible=false`, in `boosted_order` | |
| 5 | T2: `hasAfmVertical=true` + `hasAfmCampaign=true` | |
| 6 | T2: "Shop the Meal Box" near top | |
| 7 | T2: "Best $12 meals in Miami" at pinned position | |
| 8 | T1→T2: Clear position shift for both AFM carousels | |
| 9 | T3: Non-AFM NV carousels `eligible=true`, in `unboosted_order` | |
| 10 | All debug log snippets captured | |
| 11 | All screenshots/recordings captured | |
| 12 | Carousel ID → name mapping established | |

## Overall Verdict

**PENDING** — <!-- PASS / FAIL / PASS WITH NOTES -->

---

## Appendix: Raw Artifacts

### Full Pod Log Dumps
- T1: `results/t1-debug-logs.txt`
- T2: `results/t2-debug-logs.txt`
- T3: `results/t3-debug-logs-non-afm.txt`

### Screenshots
- T1: `results/t1-homepage-full.png`
- T2: `results/t2-homepage-full.png`
- Side-by-side: `results/t1-vs-t2-comparison.png`

### Recordings
- T1: `results/t1-homepage-scroll.mp4`
- T2: `results/t2-homepage-scroll.mp4`
