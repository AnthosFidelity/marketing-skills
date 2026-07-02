# Reporting and GAQL

## GAQL Resource Compatibility

**Critical:** Not all Google Ads resources can be joined in a single query.

### search_term_view
Use `segments.keyword.info.*` for keyword text and match type — **never** `ad_group_criterion.*`:

```sql
SELECT
    search_term_view.search_term,
    customer.descriptive_name,
    segments.keyword.info.text,
    segments.keyword.info.match_type,
    campaign.name,
    ad_group.name,
    metrics.clicks,
    metrics.impressions,
    metrics.conversions,
    metrics.cost_micros
FROM search_term_view
WHERE segments.date DURING LAST_7_DAYS
ORDER BY metrics.impressions DESC
```

`ad_group_criterion` is **incompatible** with `search_term_view` as the FROM resource. Selecting `ad_group_criterion.*` fields will always fail with `PROHIBITED_RESOURCE_TYPE_IN_SELECT_CLAUSE`.

### keyword_view
For keyword performance (Quality Score, bid estimates):
```sql
SELECT
    ad_group_criterion.keyword.text,
    ad_group_criterion.keyword.match_type,
    metrics.clicks,
    metrics.impressions,
    metrics.historical_quality_score
FROM keyword_view
WHERE segments.date DURING LAST_30_DAYS
```

## Cached Data (optional, when `google_ads_query_insights` is exposed)

If the MCP exposes `google_ads_query_insights`, use it for large multi-account performance queries — it reads from a local cache refreshed hourly and avoids API timeouts entirely.

### Query Performance

Call `google_ads_query_insights` — its built-in description includes the exact table name, schema, and example queries. Read the tool description first, then pass a SQL query targeting that table. Typical pattern:

```
google_ads_query_insights(
  query="SELECT date, campaign_name, SUM(cost_micros) as spend, SUM(clicks) as clicks
         FROM <table>
         WHERE customer_id = '1234567890'
           AND date >= CURRENT_DATE - INTERVAL '7 days'
         GROUP BY date, campaign_name ORDER BY date DESC"
)
```

Replace `<table>` with the table name shown in the `google_ads_query_insights` tool description. Cache is refreshed hourly — no manual sync needed.

> **If the tool returns a "no data cached" error**, check the `suggestion` field in the response — it will contain the correct workspace-specific table name. Retry the query using that suggested table name instead.

Key columns available (verify in tool description):
- **Hierarchy:** `date`, `customer_id`, `campaign_id`, `campaign_name`, `campaign_status`
- **Volume:** `impressions`, `clicks`, `cost_micros` (÷ 1,000,000 for dollars)
- **Conversions:** `conversions`, `conversions_value`, `conversion_rate`
- **Efficiency:** `ctr`, `average_cpc`, `average_cpm`

Supports standard SQL aggregations and window functions. For cross-account queries, omit the `customer_id` filter and `GROUP BY` it instead.
