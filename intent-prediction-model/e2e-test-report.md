# Intent Prediction Score Logging - E2E Test Report

**Date**: 2026-03-31
**Status**: PASS
**Pod**: `feed-service-web-group1-sandbox-rd4ae1c9-fb7996bb7-5v4vn`
**Namespace**: `feed-service-sandbox`

## Test Configuration

- DV bypass: SibylRegressor hardcoded to call model `1774285074` without experiment check
- Blending bypass: VerticalBlending checks only `score != null`, not config
- Consumer ID: hardcoded `757606047L`
- Iguazu: all gates commented out + DEBUG_IGUAZU_EMIT logging
- VerticalBlending: DEBUG_INTENT_SCORE logging

## Results

### 1. verticalIntentPredictionScore: NON-NULL (PASS)

Model `1774285074` is returning real scores. All entities show `verticalIntentPredictionScore` in the range ~15.67-15.75.

Sample log lines (verbatim):

```
DEBUG_INTENT_SCORE entity=recommended verticalIntentPredictionScore=15.746496200561523 intentMultiplier=1.0 entityScore=0.30632671385221855
DEBUG_INTENT_SCORE entity=9bc61133-ff0c-40dc-b642-3568d97ee6a7 verticalIntentPredictionScore=15.746496200561523 intentMultiplier=1.0 entityScore=0.25347599610008054
DEBUG_INTENT_SCORE entity=08ec949a-d36c-456a-bef2-707bae2f0859 verticalIntentPredictionScore=15.747407913208008 intentMultiplier=1.0 entityScore=0.1200171764267457
DEBUG_INTENT_SCORE entity=57100d88-e15a-49a8-b81d-44a253761fd1 verticalIntentPredictionScore=15.722946166992188 intentMultiplier=1.0 entityScore=0.18115988024100105
DEBUG_INTENT_SCORE entity=sponsored_carousel_boosted verticalIntentPredictionScore=15.751298904418945 intentMultiplier=1.0 entityScore=0.14134199215809717
DEBUG_INTENT_SCORE entity=21b80369-ed51-4105-abd6-b7716dd97cb9 verticalIntentPredictionScore=15.671564102172852 intentMultiplier=1.0 entityScore=0.09641412386384561
DEBUG_INTENT_SCORE entity=ff647ccb-2401-42b1-b6c0-a07619f4d013 verticalIntentPredictionScore=15.675018310546875 intentMultiplier=1.0 entityScore=0.21300108988397679
DEBUG_INTENT_SCORE entity=b8e65260-5a86-499c-89a1-2bffcbf85c1b verticalIntentPredictionScore=15.676507949829102 intentMultiplier=1.0 entityScore=0.19090245329471253
```

Scores observed across multiple consumers (50968820, 468654115) and submarkets (88, 10).

### 2. Iguazu Events: EMITTING (PASS)

`DEBUG_IGUAZU_EMIT` confirms events are being published, including `home_page_store_ranker_compact_event` which carries score_modifiers.

Sample log lines (verbatim):

```
DEBUG_IGUAZU_EMIT topic=home_page_store_ranker_compact_event eventCount=1 eventTypes=[ProtoEvent]
DEBUG_IGUAZU_EMIT topic=ads_sl_tracking_event eventCount=1 eventTypes=[ProtoEvent]
```

### 3. Model Call Errors: NONE

No errors found matching `vertical_intent`, `intentPredict`, or model ID `1774285074`.

### 4. score_modifiers Key Presence

No dedicated `vertical_intent_prediction_score` log line was found, which is expected -- we did not add DEBUG_SCORE_MODIFIERS logging. The score is set in `ScoreModifiersBuilder` and emitted via Iguazu. Snowflake validation (separate step) will confirm the key appears in the `score_modifiers` JSON column.

## Assessment

**PASS** -- The full pipeline is working end-to-end:

1. Model `1774285074` is called and returns non-null scores (~15.67-15.75)
2. `verticalIntentPredictionScore` flows through VerticalBlending to all homepage entities
3. `intentMultiplier=1.0` confirms the score is not affecting ranking (log-only mode)
4. Iguazu events including `home_page_store_ranker_compact_event` are emitting

## Next Steps

- Run Snowflake validation (`/validate-iguazu`) to confirm `vertical_intent_prediction_score` appears in `score_modifiers` column of `home_page_store_ranker_compact_event` table
- Remove all sandbox test hacks (DV bypass, blending bypass, consumer ID hardcode, Iguazu gate bypass)
- Prepare PR for review with production-ready code (experiment-gated)
