# GenAI Reranker Logging — Status Report
**Date**: 2026-03-24 05:45 UTC (2026-03-23 22:45 PST)
**Branch**: `feat/genai-reranker-logging` in feed-service

## Summary
**First successful end-to-end deployment achieved.** IguazuModule debug code is running on sandbox. Events are flowing to Kafka for `cx_cross_vertical_homepage_feed` (165 events per doortest request). Homepage loads with carousels. However, **genai_embedding and reranker fields are empty** because the test fallback code (`?: 0.42`, reranker injection) is in `services/feed-service`, not `libraries/platform` — and the devbox Makefile only compiles and patches `libraries/platform`.

## What's Working

1. **Sandbox deployed and serving traffic** — Pod `jvjkh` on sandbox `rd3987f9`
2. **IguazuModule debug code confirmed running** — `DEBUG_IGUAZU_ENTER/GENERATE/COUNT/SENT` all appearing in pod logs
3. **Events flowing to Kafka** — `cx_cross_vertical_homepage_feed`: 165 events sent for doortest tenant
4. **Sandbox bypass working** — `val isSandboxEnv = false` allows Iguazu events in sandbox
5. **Event sending re-enabled** — `proxyEventSender.send(event)` is active
6. **Homepage loads** — Carousels visible: "Fastest", "Under $1 delivery fee", "National favorites", etc.

## Root Cause: genai/reranker fields empty

The devbox Makefile target `build-remote-devbox-server` only compiles `libraries/platform` and patches the bootJar with the platform JAR. Changes in other modules are NOT deployed:

| File | Module | Deployed? | Change |
|------|--------|-----------|--------|
| `IguazuModule.kt` | `libraries/platform` | YES | Debug logging, sandbox bypass, event sending |
| `StoreCarouselService.kt` | `services/feed-service` | **NO** | `embeddingScore ?: 0.42` fallback |
| `HomepageProgrammaticProductServiceUtil.kt` | `services/feed-service` | **NO** | `?: 0.42` fallback |
| `StoreCarouselDataAdapterUtil.kt` | `services/feed-service` | **NO** | reranker alpha/beta injection |
| `HomepageRequestToContext.kt` | `pipelines/homepage` | **NO** | Consumer ID override |

The `DEBUG_IGUAZU_SUMMARY` logs confirm: `genai_count=0, reranker_count=0, cxgen_count=0` — Content Systems returns null for embedding scores in this submarket, and our test fallback code isn't deployed.

## Build Status
- **BUILD SUCCESSFUL** on all runs since v7
- Build takes ~3s with cached platform library
- Service startup takes ~52s after build
- Devbox exits immediately after "Service is ready" — service survives on pod

## Key Discovery: devbox lifecycle
- `devbox run` syncs files, builds, patches bootJar, restarts service, waits for ready, then **exits**
- Service continues running on pod after devbox exits (on new pods like `jvjkh`)
- On older pods, the restarted service may die when devbox SSH disconnects
- Previous 502 errors were from testing on old pods where the service died after devbox exit

## What's Done

### Code Changes (all on branch, uncommitted local-only changes marked)

1. **`IguazuModule.kt`** — Debug logging + sandbox bypass (uncommitted) — **DEPLOYED**
   - `val isSandboxEnv = false` hardcode to bypass sandbox Iguazu gating
   - DEBUG_IGUAZU_ENTER/SKIP/GENERATE/COUNT/SENT log lines
   - Detailed per-event inspection for `cx_cross_vertical_homepage_feed` topic
   - Logs genai_embedding counts, reranker counts, cxgen event details
   - Events ARE sent to Kafka (re-enabled after accidental disable)

2. **`StoreCarouselService.kt:792`** — Test fallback (uncommitted) — **NOT DEPLOYED**
   - `embeddingScore = ...embeddingScore ?: 0.42`

3. **`HomepageProgrammaticProductServiceUtil.kt:225`** — Test fallback (uncommitted) — **NOT DEPLOYED**
   - Same `?: 0.42` fallback

4. **`StoreCarouselDataAdapterUtil.kt:663-672`** — Test reranker values (uncommitted) — **NOT DEPLOYED**
   - Injects `reranker_alpha: 0.5, reranker_beta: 1.5`

5. **`HomepageRequestToContext.kt`** — Consumer ID override (uncommitted) — **NOT DEPLOYED**
   - `return 757606047L`

6. **`SpeedCategoryFilterProductService.kt`** — Cache buster annotation (uncommitted)
   - `@file:Suppress("unused")` to bust stale Gradle remote cache

7. **Committed code** (on branch, pushed):
   - `genai_embedding_similarity_score` LoggedValue enum in ContainerEventsGenerator
   - Score modifier logging in IguazuEventUtil
   - `embeddingScore` field on StoreCarouselItem data class
   - carousel_details tracking payload plumbing

## Current State
- Sandbox: `rd3987f9`
- Pod: `feed-service-web-group1-sandbox-rd3987f9-57cd8b88c-jvjkh`
- Service: RUNNING with patched IguazuModule
- Events: Flowing to Kafka (165 events per doortest homepage request)
- Homepage: Loading with carousels

## Next Steps (in order)

1. **Modify Makefile** to compile and patch additional modules (`services/feed-service`, `pipelines/homepage`) — OR move test fallback logic into IguazuModule (platform library) where the event generation happens
2. **Re-deploy** with test fallbacks compiled
3. **Verify genai_count > 0** in DEBUG_IGUAZU_SUMMARY logs
4. **Query Snowflake** to verify `genai_embedding_similarity_score` and reranker data appear
5. **Remove test fallbacks** before final commit
6. **Clean up** cache buster annotations

## Alternative: Skip test fallbacks

Since the committed code changes add the `genai_embedding_similarity_score` field plumbing, and the IguazuModule confirms events flow end-to-end, we could:
1. Verify in Snowflake that the field COLUMN exists (even if values are null for this submarket)
2. Merge the PR as-is — the real data will come from production submarkets where Content Systems returns real embedding scores
3. This avoids the need to hack the Makefile for a one-time test

## Key Learnings

- `build-remote-devbox-server` Makefile target only compiles `libraries/platform` — NOT the full service
- Devbox exits after "Service is ready" — service survives on new pods
- On old recycled pods, the service may die when devbox exits (causing 502s)
- Teleport connections drop intermittently — `namespaces not found` errors
- Always wait for devbox to finish before testing (don't test while service starting)
- Pod names change between devbox runs — always get fresh pod name from devbox output
