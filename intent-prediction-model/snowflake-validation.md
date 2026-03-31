# Snowflake Validation: vertical_intent_prediction_score

**Date**: 2026-03-31
**Status**: BLOCKED -- no events to validate
**Consumer ID**: 757606047
**Table**: `IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE`

## Result: No Snowflake Validation Possible

Snowflake validation could not be performed for two independent reasons:

### Reason 1: No events reached Snowflake (Iguazu bypass compile failure)

The `:platform:compileKotlin` task failed during sandbox build. The `IguazuModule.kt` change (`val isSandboxEnv = false`) lives in the `:platform` module. Because compilation failed, the bypass was never applied to the running JAR. The E2E test report confirms: `ENABLE_SANDBOX_IGUAZU` environment variable lookup is happening (the default code path), meaning the hardcoded bypass is NOT active.

Without the bypass, sandbox Iguazu gating blocks all events from reaching Snowflake. Even if homepage requests had been processed, no events would have been published.

### Reason 2: No traffic reached the sandbox pod

The E2E test (02:47-02:49 UTC on 2026-03-31) confirmed zero homepage gRPC requests reached the sandbox pod during the browser session. Root cause: `devbox run web-group1-remote` synced from the main checkout (on `master`), not the worktree containing the feature branch code. The pod was running `master` code, not `feat/intent-prediction-score-logging`.

### Reason 3: No Snowflake tooling in this environment

No Snowflake CLI (`snowsql`, `snow`), Python connector, or MCP tool is available in the current environment. Even if events had landed, querying would require manual execution outside this agent.

## Queries to Run Manually

When Snowflake access is available (e.g., via Snowsight UI or a configured CLI), run these queries after a successful sandbox test:

### Query 1: Check if any events exist for our consumer
```sql
SELECT COUNT(*) AS CNT, MAX(IGUAZU_PARTITION_HOUR) AS LATEST_HOUR
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
WHERE IGUAZU_PARTITION_DATE = '<test-date>'
  AND IGUAZU_PARTITION_HOUR BETWEEN <hour-1> AND <hour+2>
  AND CONSUMER_ID = 757606047;
```

### Query 2: Check for the new score modifier
```sql
SELECT
    e.REQUEST_ID,
    e.FACET_ID,
    e.FACET_TYPE,
    sm.value:name::STRING AS modifier_name,
    sm.value:value::DOUBLE AS modifier_value
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE e,
    LATERAL FLATTEN(input => PARSE_JSON(e.SCORE_MODIFIERS)) sm
WHERE e.IGUAZU_PARTITION_DATE = '<test-date>'
  AND e.IGUAZU_PARTITION_HOUR BETWEEN <hour-1> AND <hour+2>
  AND e.CONSUMER_ID = 757606047
  AND sm.value:name::STRING = 'vertical_intent_prediction_score'
ORDER BY e.FACET_VERTICAL_POSITION
LIMIT 20;
```

### Query 3: All score modifiers overview
```sql
SELECT
    sm.value:name::STRING AS modifier_name,
    COUNT(*) AS cnt,
    AVG(sm.value:value::DOUBLE) AS avg_val
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE e,
    LATERAL FLATTEN(input => PARSE_JSON(e.SCORE_MODIFIERS)) sm
WHERE e.IGUAZU_PARTITION_DATE = '<test-date>'
  AND e.IGUAZU_PARTITION_HOUR BETWEEN <hour-1> AND <hour+2>
  AND e.CONSUMER_ID = 757606047
GROUP BY modifier_name
ORDER BY cnt DESC;
```

### Query 4: Cross-reference raw score with multiplier
```sql
SELECT e.FACET_ID,
    MAX(CASE WHEN sm.value:name::STRING = 'vertical_intent_prediction_score' THEN sm.value:value::DOUBLE END) AS raw_score,
    MAX(CASE WHEN sm.value:name::STRING = 'vertical_intent_multiplier' THEN sm.value:value::DOUBLE END) AS multiplier
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE e,
    LATERAL FLATTEN(input => PARSE_JSON(e.SCORE_MODIFIERS)) sm
WHERE e.IGUAZU_PARTITION_DATE = '<test-date>'
  AND e.IGUAZU_PARTITION_HOUR BETWEEN <hour-1> AND <hour+2>
  AND e.CONSUMER_ID = 757606047
  AND sm.value:name::STRING IN ('vertical_intent_prediction_score', 'vertical_intent_multiplier')
GROUP BY e.FACET_ID;
```

## Prerequisites for a Successful Retry

All of the following must be true before re-attempting Snowflake validation:

1. **Fix `:platform:compileKotlin`** -- the IguazuModule.kt change must compile cleanly
2. **Sync from worktree** -- run `devbox run web-group1-remote` from inside the worktree directory, OR checkout the feature branch on the main feed-service directory
3. **Verify pod is running feature code** -- check startup logs for evidence of the feature branch
4. **Apply Iguazu bypass** -- `val isSandboxEnv = false` in `IguazuModule.kt` must be active
5. **Hardcode consumer ID** -- `return 757606047L` in `HomepageRequestToContext.kt`
6. **Browse homepage** -- visit `https://www.doordashtest.com/` with test credentials
7. **Wait 3-5 minutes** -- events need propagation time to land in Snowflake
8. **Have Snowflake access** -- either configure `snowsql`/`snow` CLI, the Python connector, or use the Snowsight web UI

## Expected Outcome When Working

- Query 1 returns CNT > 0 (events exist)
- Query 2 returns rows with `modifier_name = 'vertical_intent_prediction_score'` and values in [0.0, 1.0]
- Query 3 shows the new modifier alongside existing modifiers like `vertical_intent_multiplier`
- Query 4 shows both raw score and multiplier coexisting on the same facets

## Acceptance Criteria

| Check | Pass Condition |
|---|---|
| Events exist in Snowflake | CNT > 0 for test consumer/date |
| New modifier present | At least 1 row with `vertical_intent_prediction_score` |
| Values valid | All values >= 0, no NaN/Infinity |
| Coexists with multiplier | Both `vertical_intent_prediction_score` and `vertical_intent_multiplier` on same facets |
| No regression | All prior modifiers maintain expected coverage |
