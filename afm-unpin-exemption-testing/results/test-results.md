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
| PR branch | `syd/afm-unpin-exempt-campaigns` |
| PR HEAD | `b49946d46c8` ("Register enable_afm_unpin_exemption DV in DiscoveryExperimentManager") |
| Sandbox pod | `feed-service-web-group1-sandbox-rd36d002-5748ffdb8f-pz4k7` |
| Sandbox namespace | `feed-service-sandbox` |
| Sandbox routing URL | `https://www.doordashtest.com/developer/sandbox/W3sic2VydmljZSI6ImZlZWQtc2VydmljZSIsImFwcCI6IndlYi1ncm91cDEiLCJob3N0IjoiZmVlZC1zZXJ2aWNlLXdlYi1ncm91cDEtc2FuZGJveC1yZDM2ZDAwMiIsInBvcnQiOiI1MDA1MSJ9XQ==` |
| feed-service worktree | `.claude/worktrees/afm-unpin-test` (instrumentation source) |
| feed-service main checkout | `syd/afm-unpin-exempt-campaigns` (sync source — workspace `rd36d002`) |

---

## Setup Log

| Step | Status | Notes |
|------|--------|-------|
| feed-service worktree created | DONE | Branch `pr-62270` at commit `b8600da458b` (v1) |
| Consumer ID hardcoded | DONE | `HomepageRequestToContext.kt:1038` → `return 757606047L` |
| Debug instrumentation v1 | DONE | 4 log points on old DV mechanism (`PersonalizationRuntimeUtil`). All firing. |
| Sandbox synced v1 | DONE | Service ready at `rd36d002`. Debug logs confirmed in pod. |
| **PR updated (4 commits)** | DONE | DV moved from `PersonalizationRuntimeUtil` → `DiscoveryExperimentManager.Manifest`. `isExemptAfmCarousel` now takes `experimentMap`. DV values are `treatment`/`control`/`absent` (not `true`/`false`). |
| Pulled latest + rebased instrumentation | DONE | Main checkout at `b49946d46c8`. Debug log points rewritten for new DV mechanism. Consumer ID re-hardcoded. |
| Local compile check | DONE | `./gradlew compileKotlin` BUILD SUCCESSFUL (domain-util) |
| Sandbox respawn + resync | DONE | New pod `pz4k7`. BUILD SUCCESSFUL in 26m 51s. Service ready. |
| `[AFM_DEBUG]` v2 confirmed in pod | DONE | All 4 log points firing. `DV=control` (expected — default). `isExemptAfmCarousel` detail log correctly suppressed when DV=control (early return). `unpin_check` shows `nvEligible`/`afmExempt`/`willUnpin` fields. |

---

## Carousel ID Mapping

> Established during T1. Note: `cxgen:cxgen:*` IDs are CxGen-generated item carousels. The AFM carousels contain vertical `100322` (AFFORDABLE_MEALS_VERTICAL_ID).

| Carousel ID | Display Name | Type | Has Vertical 100322? |
|-------------|-------------|------|---------------------|
| `e25be2d7-3912-43b4-9680-e4ad2928a152` | Best $12 meals in Miami | Store | YES |
| `cxgen:cxgen:9` | Shop the Meal Box collection | Item (CxGen) | YES |
| `288e689e-614c-4035-a90b-781be6aa6ac5` | *(NV carousel — non-AFM)* | Store | NO |
| `cd25f777-0e1a-41aa-ba7b-97919f0606bd` | *(NV carousel — non-AFM)* | Store | NO |

---

## Test 1: Baseline — DV OFF

**Executed**: 2026-03-27 ~17:02 UTC
**DV `enable_afm_unpin_exemption`**: `control` (default — not in treatment)
**Consumer**: `1125900615344792` (doortest tenant; feed logic uses hardcoded `757606047L` for personalization)

### Debug Log Output
```
[AFM_DEBUG] enable_afm_unpin_exemption DV=control
[AFM_DEBUG] isEligibleForUnpin: carouselId=288e689e nvEligible=true (NV, non-AFM)
[AFM_DEBUG] unpin_check: carouselId=288e689e nvEligible=true afmExempt=false willUnpin=true
[AFM_DEBUG] isEligibleForUnpin: carouselId=cd25f777 nvEligible=true (NV, non-AFM)
[AFM_DEBUG] unpin_check: carouselId=cd25f777 nvEligible=true afmExempt=false willUnpin=true
[AFM_DEBUG] isEligibleForUnpin: carouselId=e25be2d7 verticals=[100322,...] nvEligible=true (AFM!)
[AFM_DEBUG] unpin_check: carouselId=e25be2d7 nvEligible=true afmExempt=false willUnpin=true
[AFM_DEBUG] isEligibleForUnpin: carouselId=cxgen:cxgen:9 verticals=[100322,...] nvEligible=true (AFM!)
[AFM_DEBUG] unpin_check: carouselId=cxgen:cxgen:9 nvEligible=true afmExempt=false willUnpin=true
[AFM_DEBUG] boosted_order=[4e12d866, cxgen:0-8, sponsored_carousel_boosted, cc900279]
[AFM_DEBUG] unboosted_order=[other, recommended, trending, ..., 288e689e, cd25f777, e25be2d7, cxgen:cxgen:9]
```

### Carousel Position Map (visible on homepage)
| Position | Display Name | Boost Status |
|----------|-------------|--------------|
| 1 | Your past orders | N/A (non-boosting) |
| 2 | Shop the Meal Box collection (`cxgen:cxgen:9`) | **UNBOOSTED** (unpinned) |
| 3 | Best $12 meals in Miami (`e25be2d7`) | **UNBOOSTED** (unpinned) |
| 4 | Savory poke bowls | CxGen |
| 5 | Top finds for you | CxGen |
| 6 | Fresh salads | CxGen |
| 7 | Mediterranean protein bowls | CxGen |
| 8 | Thai protein bowls | CxGen |
| 9 | Gourmet sandwiches | CxGen |
| 10 | Japanese sushi rolls | CxGen |
| 11 | Top pickup options nearby | Store |
| 12 | Quick essentials nearby | Store |
| 13 | Convenience & drugstores | Store |

### Screenshots
- Full page: `results/t1-homepage-sandbox.png`

### Pass Criteria
- [x] Debug logs show `DV=control` ✅
- [x] AFM carousels show `nvEligible=true` and `afmExempt=false` (DV off → early return, no exemption) ✅
- [x] AFM carousel IDs appear in `unboosted_order` ✅ (`e25be2d7` and `cxgen:cxgen:9`)
- [x] Screenshot captured ✅

### Analysis
- **DV=control confirmed**: `isExemptAfmCarousel` returns `false` immediately (early return), so no detail log fires. Correct behavior.
- **4 NV carousels unpinned**: `288e689e`, `cd25f777`, `e25be2d7` (AFM), `cxgen:cxgen:9` (AFM). All moved to `unboosted_order`.
- **AFM carousels visible but not pinned**: "Shop the Meal Box collection" at position 2, "Best $12 meals in Miami" at position 3 — these are in the NV unpin zone, not boosted.
- **No `isExemptAfmCarousel` detail log** — expected, because DV=control triggers early return before the vertical/campaign check.
- **Consumer ID note**: Pod metadata shows `1125900615344792` (doortest account), but feed logic uses hardcoded `757606047L` for personalization. Debug logs correctly fire for this consumer's requests.

---

## Test 2: DV ON — AFM Pinned

**Executed**: *pending*
**DV `enable_afm_unpin_exemption`**: `treatment` (in experiment map)

### Debug Log Output
```
<!-- Raw [AFM_DEBUG] output -->
```

### AFM Detection Detail
```
<!-- isExemptAfmCarousel log lines showing campaign tags -->
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
- [ ] Debug logs show `DV=treatment`
- [ ] AFM carousels show `afmExempt=true` and `willUnpin=false`
- [ ] `isExemptAfmCarousel` log shows `hasAfmVertical=true` + `hasAfmCampaign=true`
- [ ] AFM carousel IDs in `boosted_order`
- [ ] "Shop the Meal Box" near top of homepage
- [ ] "Best $12 meals in Miami" at pinned position
- [ ] Visual position shift from T1
- [ ] Other NV carousels show `nvEligible=true` and `afmExempt=false`

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
| 1 | T1: `DV=control/absent` and `afmExempt=false` confirmed | |
| 2 | T1: AFM carousels `nvEligible=true`, `willUnpin=true`, in `unboosted_order` | |
| 3 | T2: `DV=treatment` and `afmExempt=true` confirmed | |
| 4 | T2: AFM carousels `willUnpin=false`, in `boosted_order` | |
| 5 | T2: `hasAfmVertical=true` + `hasAfmCampaign=true` in `isExemptAfmCarousel` log | |
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
