# E2E Test Report: Intent Prediction Score Logging

**Date**: 2026-03-31
**Test timestamp (UTC)**: 2026-03-31T16:41:36Z (routing activation), 2026-03-31T16:43:00Z (homepage request with full pipeline)
**Pod**: `feed-service-web-group1-sandbox-rd4ae1c9-fb7996bb7-5v4vn`
**Namespace**: `feed-service-sandbox`
**Trace ID**: `577219005cc12ff4601b01abcb6bde37`
**Consumer ID observed**: `1125900615344792` (NOT `757606047` -- see findings)

---

## Summary

| Criteria | Result |
|----------|--------|
| Homepage loads in sandbox | PASS |
| DEBUG_IGUAZU_EMIT fires | PASS (28 events across 11 topics) |
| DEBUG_INTENT_SCORE fires | PASS (29 entities logged) |
| DEBUG_SCORE_MODIFIERS fires | PASS (293 entries logged) |
| `vertical_intent_prediction_score` in score_modifiers | FAIL -- not present |
| `verticalIntentPredictionScore` non-null in intent logs | FAIL -- null for all entities |
| Consumer ID hardcoded to 757606047 | FAIL -- request shows `1125900615344792` |

**Overall: PARTIAL PASS** -- Debug logging infrastructure works end-to-end. The intent prediction score itself is null, meaning the ML model call is not returning a score for this consumer/submarket. The consumer ID hardcode did not take effect.

---

## DEBUG_IGUAZU_EMIT (28 events)

Topics emitted:
```
 10 cx_home_page_feed_carousel_event
  7 home_page_store_ranker_compact_event
  2 feed_service_grpc_performance
  2 ads_sl_tracking_event
  1 vertical_entry_point_event
  1 smart_suggestion_event_load
  1 reorder_carousel_create_event
  1 home_page_pagination_event
  1 cx_home_page_feed_carousel_capping_event
  1 cx_cross_vertical_homepage_feed
  1 announcement_collection_event
```

Key topic `home_page_store_ranker_compact_event` fired 7 times -- this is the topic that carries `score_modifiers` to Snowflake.

---

## DEBUG_INTENT_SCORE (29 entities, all null)

Representative lines:
```
DEBUG_INTENT_SCORE entity=recommended verticalIntentPredictionScore=null intentMultiplier=1.0 intentPredictionModel= entityScore=0.01399810123257339
DEBUG_INTENT_SCORE entity=fastest_near_you_cgna_homepage verticalIntentPredictionScore=null intentMultiplier=1.0 intentPredictionModel= entityScore=0.008568756478773476
DEBUG_INTENT_SCORE entity=07e18b95-82af-42bc-b71d-ba1479794885 verticalIntentPredictionScore=null intentMultiplier=1.0 intentPredictionModel= entityScore=0.0035075940538031106
DEBUG_INTENT_SCORE entity=08ec949a-d36c-456a-bef2-707bae2f0859 verticalIntentPredictionScore=null intentMultiplier=1.0 intentPredictionModel= entityScore=0.014201020239852369
DEBUG_INTENT_SCORE entity=2393d0ca-c7d7-46a4-a158-8f46ba4e7da0 verticalIntentPredictionScore=null intentMultiplier=1.0 intentPredictionModel= entityScore=0.01090568086089172
```

**All 29 entities show `verticalIntentPredictionScore=null` and `intentPredictionModel=` (empty string).**

This means the ML model prediction call is not returning a score. Possible causes:
1. The intent prediction model (ID `1774285074`) is not deployed/available in the sandbox environment
2. The model call is behind a DV flag that is off for this consumer
3. The consumer ID `1125900615344792` is not in the model's treatment group

---

## DEBUG_SCORE_MODIFIERS (293 entries)

All entries show the same 3 fields:
```
DEBUG_SCORE_MODIFIERS vertical_intent_multiplier=1.0 calibration_multiplier=1.0 vertical_boost_weight={1.0|1.25|3.75|5.0}
```

**`vertical_intent_prediction_score` does NOT appear in any SCORE_MODIFIERS log line.** This confirms the score is not being written to the Iguazu event's `score_modifiers` map -- expected since the score is null upstream.

---

## Consumer ID Issue

The request logs show `consumer_id: 1125900615344792` instead of the expected hardcoded `757606047`. This means either:
1. The `getConsumerIdFromRequest` hardcode was not in the deployed code
2. The homepage request goes through a different consumer ID resolution path
3. The code sync (`devbox run web-group1-remote`) did not include the hardcode file

**Action needed**: Verify the hardcode is present in the synced code on the pod. Run:
```bash
kubectl -n feed-service-sandbox exec feed-service-web-group1-sandbox-rd4ae1c9-fb7996bb7-5v4vn -- find / -name "HomepageRequestToContext*" 2>/dev/null
```

---

## Next Steps

1. **Fix consumer ID**: Verify `HomepageRequestToContext.kt` hardcode was synced to pod
2. **Investigate null score**: Check if model `1774285074` is accessible from sandbox, check DV flag gating
3. **Re-run test** after fixing consumer ID to see if model returns scores for consumer `757606047`
4. **Snowflake validation**: Once scores are non-null, verify they appear in `home_page_store_ranker_compact_event` Snowflake table
