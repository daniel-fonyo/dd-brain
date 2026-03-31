# Snowflake Validation: vertical_intent_prediction_score

**Date**: 2026-03-31
**Status**: BLOCKED -- no Snowflake query capability
**Consumer ID**: 757606047
**Table**: `IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE`

## Result: Cannot Execute Queries

Attempted validation at ~09:00 UTC on 2026-03-31 (after 8-minute wait for browser test + Snowflake propagation). However, no Snowflake query tool is available in this session:

- **Snowflake MCP server**: NOT configured in `~/.claude/mcp.json` (only `playwright` is present)
- **snowsql CLI**: not installed
- **snowflake Python connector**: not installed

The previous validation doc referenced a `snowflake` MCP server, but it is not currently in the MCP config.

## Action Required

To run these queries, either:

1. **Add the Snowflake MCP server** back to `~/.claude/mcp.json` (e.g. `snowflake-labs-mcp` via `uvx` with auth env vars)
2. **Install snowsql** (`brew install --cask snowflake-snowsql`)
3. **Run manually** in a Snowflake web UI / worksheet

## Queries to Execute

### Query 1: Check events exist
```sql
SELECT COUNT(*) AS cnt, MIN(IGUAZU_PARTITION_HOUR) AS min_hour, MAX(IGUAZU_PARTITION_HOUR) AS max_hour
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE
WHERE IGUAZU_PARTITION_DATE = '2026-03-31'
  AND IGUAZU_PARTITION_HOUR BETWEEN 7 AND 10
  AND CONSUMER_ID = 757606047;
```

### Query 2: Check for new score modifier
```sql
SELECT sm.value:name::STRING AS modifier_name, sm.value:value::DOUBLE AS modifier_value, e.FACET_ID, e.FACET_TYPE
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE e,
    LATERAL FLATTEN(input => PARSE_JSON(e.SCORE_MODIFIERS)) sm
WHERE e.IGUAZU_PARTITION_DATE = '2026-03-31'
  AND e.IGUAZU_PARTITION_HOUR BETWEEN 7 AND 10
  AND e.CONSUMER_ID = 757606047
  AND sm.value:name::STRING = 'vertical_intent_prediction_score'
ORDER BY e.FACET_VERTICAL_POSITION
LIMIT 20;
```

### Query 3: All score modifiers summary
```sql
SELECT sm.value:name::STRING AS modifier_name, COUNT(*) AS cnt, AVG(sm.value:value::DOUBLE) AS avg_val
FROM IGUAZU.SERVER_EVENTS_PRODUCTION.CX_CROSS_VERTICAL_HOME_PAGE_FEED_ICE e,
    LATERAL FLATTEN(input => PARSE_JSON(e.SCORE_MODIFIERS)) sm
WHERE e.IGUAZU_PARTITION_DATE = '2026-03-31'
  AND e.IGUAZU_PARTITION_HOUR BETWEEN 7 AND 10
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
WHERE e.IGUAZU_PARTITION_DATE = '2026-03-31'
  AND e.IGUAZU_PARTITION_HOUR BETWEEN 7 AND 10
  AND e.CONSUMER_ID = 757606047
  AND sm.value:name::STRING IN ('vertical_intent_prediction_score', 'vertical_intent_multiplier')
GROUP BY e.FACET_ID
LIMIT 20;
```

### Broader fallback queries (if no results above)
Try `IGUAZU_PARTITION_DATE = '2026-03-30'` and hours 7-10 in case of timezone offset.

## Acceptance Criteria

| Check | Pass Condition |
|---|---|
| Events exist in Snowflake | CNT > 0 for test consumer/date |
| New modifier present | At least 1 row with `vertical_intent_prediction_score` |
| Values valid | All values >= 0, no NaN/Infinity |
| Coexists with multiplier | Both `vertical_intent_prediction_score` and `vertical_intent_multiplier` on same facets |
| No regression | All prior modifiers maintain expected coverage |
