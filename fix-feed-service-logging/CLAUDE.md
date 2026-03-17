# Fix Feed-Service Logging

## Goal
Design and implement proper logging for the feed-service homepage pipeline:
1. Define a Snowflake table schema (`CX_HOMEPAGE_STORE_LOADS`) that captures one row per store loaded on a `getHomepageFeed` request
2. Extend the existing Iguazu event (`cx_home_page_feed` / `HomePageFeedEvent`) with missing fields
3. Update `LegacyHomePageIguazuEventUtil` to populate those fields

## Key Files in feed-service
- Proto: `services/feed-service/build/extracted-include-protos/main/feed_service/events.proto` — `HomePageFeedEvent`
- Event generator: `libraries/domain-util/src/main/kotlin/com/doordash/consumer/feed/domainUtil/iguazu/LegacyHomePageIguazuEventUtil.kt`
- Emit point: `pipelines/homepage/src/main/kotlin/com/doordash/consumer/pipelines/homepage/DefaultHomePageResponseSerializer.kt`
- Iguazu infrastructure: `libraries/platform/src/main/kotlin/com/doordash/consumer/feed/platform/iguazu/IguazuModule.kt`
