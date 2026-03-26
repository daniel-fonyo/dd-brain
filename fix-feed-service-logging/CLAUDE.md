# Fix Feed-Service Logging — Investigation Context

## What This Directory Is

Investigation and impact analysis of the store carousel Iguazu events regression (Jan 21 – Mar 10, 2026). ~32B events lost over 49 days due to a broken facet ID format in the Andromeda code path.

## Context Files

- `context/Store Carousel Iguazu Events Regression.md` — Full regression doc: what happened, why, how it was fixed, DV timeline, daily breakdown
- `context/impact-report.md` — CX user impact analysis: before/after, treatment/control, CTR-based GOV estimation, findings (no measurable CX impact detected)
- `context/order-impact-analysis.md` — SQL queries (1–7) for measuring order rate, CTR, and GOV impact
- `context/impact-analysis-segments.md` — Stable DV windows per platform for clean comparisons
- `context/discovery-data-sources.md` — Snowflake table reference: server events, client events, join keys, business vertical IDs

## Key Files in feed-service

- Event generator: `libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/iguazu/IguazuEventUtil.kt`
- Emit point (realtime): `libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/serializer/SerializerHelperUtil.kt`
- Emit point (legacy): `pipelines/homepage/src/main/kotlin/com/doordash/consumer/pipelines/homepage/DefaultHomePageResponseSerializer.kt`
- Proto: `build/extracted-include-protos/main/feed_service/events.proto` — `HomePageFeedEvent`, `CrossVerticalHomePageFeedEvent`
- Iguazu infra: `libraries/platform/src/main/kotlin/com/doordash/consumer/feed/platform/iguazu/IguazuModule.kt`
- Fix PR: https://github.com/doordash/feed-service/pull/60973
