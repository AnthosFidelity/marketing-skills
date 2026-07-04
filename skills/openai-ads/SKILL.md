---
name: openai-ads
description: Plan and manage OpenAI Ads (ChatGPT ads) campaigns end-to-end via the Hyper MCP — API-key auth, account discovery, geo targeting, image upload, chat_card and product-feed creatives, custom audiences, conversion tracking, status flow, and insights, with integer-micros money values. Use when the user wants to launch OpenAI ads, ChatGPT ads, chat card ads, manage OpenAI ad groups or audiences, or pull OpenAI Ads insights.
---

# OpenAI Ads Campaigns

Strategic guide for managing the OpenAI Advertiser API. All operations go
through `https://api.ads.openai.com/v1`.

## Requirements

- **Hyper MCP installed and connected.** [https://app.hyperfx.ai/mcp](https://app.hyperfx.ai/mcp)
- **OpenAI Ads integration connected** (an OpenAI Ads API key, scoped to one ad account) at [https://app.hyperfx.ai/integrations](https://app.hyperfx.ai/integrations).

If `openai_ads_ad_accounts_get` is not in the tool list, stop and tell the user to enable Hyper MCP and connect OpenAI Ads. After connecting, `openai_ads_health_check()` verifies the key — if `connected=false`, the API key is missing, invalid, or expired.

## Out of scope — defer to other skills

- **Creative generation** (ad imagery, copy) → [`ad-creative-generation`](../ad-creative-generation) / [`image-generation`](../image-generation).
- **Cross-platform campaign launches** → use this skill for OpenAI Ads, then invoke `meta-ads` / `google-ads` separately.

## Critical Rules

> **CRITICAL**: Auth is bearer API-key auth, not OAuth. One Ads API key is
> scoped to exactly one ad account. There is no `list ad accounts` endpoint;
> `openai_ads_ad_accounts_get` returns the connected account for that key.

> **CRITICAL**: All money values on inputs are **integer micros**.
> `$1.00 = 1_000_000` micros. `$50/day = 50_000_000`. The
> `daily_spend_limit_micros` minimum is `1_000_000` ($1). The ad group
> `max_bid_micros` is capped at `100_000_000` ($100). Insights responses use
> plain floats in account currency, not micros.

> **CRITICAL**: Always create campaigns, ad groups, and ads with
> `status="paused"`. Surface what was created to the user, then activate
> using the dedicated activate endpoint after approval.

> **CRITICAL**: Ads have `review_status`. New ads enter `in_review` and will
> not serve until `review_status="approved"`, even if `status="active"`.

> **CRITICAL**: `chat_card` creatives require `target_url` and `file_id`.
> Upload the image first via `openai_ads_images_upload`, then pass the returned
> `file_id` to `openai_ads_create`. PNG, 1024x1024, <= 1 MB.

> **IMPORTANT**: Creative types are `chat_card` and `product_ad_template`.
> Product-ad templates get image and destination URL from the selected product
> feed item, so they do not require `file_id` or `target_url`.

> **IMPORTANT**: Campaign `bidding_type` can be `impressions` or `clicks`.
> Ad group `billing_event_type` can be `impression` or `click`.

## Tool surface

| Job | Tools |
| --- | --- |
| Account | `openai_ads_ad_accounts_get`, `openai_ads_update_ad_account`, `openai_ads_activate_ad_account`, `openai_ads_pause_ad_account`, `openai_ads_health_check` |
| Campaigns | `openai_ads_campaigns_create`, `openai_ads_campaigns_get`, `openai_ads_campaigns_list`, `openai_ads_campaigns_update`, `openai_ads_campaigns_pause`, `openai_ads_campaigns_activate`, `openai_ads_campaigns_archive` |
| Ad groups | `openai_ads_ad_groups_create`, `openai_ads_ad_groups_get`, `openai_ads_ad_groups_list`, `openai_ads_ad_groups_update`, `openai_ads_ad_groups_pause`, `openai_ads_ad_groups_activate`, `openai_ads_ad_groups_archive` |
| Ads | `openai_ads_create`, `openai_ads_get`, `openai_ads_list`, `openai_ads_update`, `openai_ads_pause`, `openai_ads_activate`, `openai_ads_archive` |
| Images & targeting | `openai_ads_images_upload`, `openai_ads_search_geo_locations` |
| Audiences | `openai_ads_create_custom_audience`, `openai_ads_create_custom_audience_upload`, `openai_ads_get_custom_audience`, `openai_ads_list_custom_audiences`, `openai_ads_archive_custom_audience` |
| Conversions | `openai_ads_create_conversion_pixel`, `openai_ads_create_conversion_api_key`, `openai_ads_create_conversion_event_setting`, `openai_ads_list_conversion_event_settings`, `openai_ads_get_conversion_insights` |
| Insights | `openai_ads_account_insights_get`, `openai_ads_campaign_insights_get`, `openai_ads_ad_group_insights_get`, `openai_ads_insights_get` |
| Cache snapshot | `openai_ads_cache`, `openai_ads_caches_get`, `openai_ads_caches_refresh` |

## Phase 1: Account Discovery

Run these after connect:

```text
openai_ads_ad_accounts_get()
openai_ads_campaigns_list(limit=100)
openai_ads_list_custom_audiences(limit=100)
openai_ads_list_conversion_event_settings(limit=100)
```

The connect-time context builder may have already populated a cached snapshot.
Prefer:

```text
openai_ads_caches_get()
```

If `success=False` because the cache is empty, refresh once:

```text
openai_ads_caches_refresh()
```

## Phase 2: Plan and Confirm

Before creating anything, confirm with the user:

- Objective in plain language.
- Daily and/or lifetime budget in account currency.
- Target geos: simple country codes like `["US", "GB"]`, or location IDs from `openai_ads_search_geo_locations`.
- Any custom audiences to include or exclude.
- Whether this is a normal `chat_card` campaign or a product-feed campaign.
- Headline (`title`, <= 50 chars), body (<= 100 chars), and click-through URL for `chat_card`.
- One image asset for `chat_card`, either a public URL or a base64 blob.
- Max bid in micros (`max_bid_micros`; for example `2_000_000`).
- Optional `context_hints`, short natural-language phrases that describe when the ad should show.
- Optional conversion event setting IDs to attach to the campaign.

If anything is missing, ask. Do not invent budgets, geos, or copy.

## Geo Targeting

Use `targeting_country_codes` for country-level targeting. For regions and
DMAs, search locations first and pass the returned IDs:

```text
openai_ads_search_geo_locations(q="San Francisco", limit=5)

openai_ads_campaigns_create(
    name="Hyper - West Coast Test",
    status="paused",
    daily_spend_limit_micros=50_000_000,
    targeting_location_ids=["2000043", "3000194"],
    bidding_type="clicks",
)
```

Do not pass geo exclusions. The current API supports included locations and
custom audience exclusions, but not location exclusions.

## Standard Chat Card Flow

```text
openai_ads_campaigns_create(
    name="Hyper - US Test",
    status="paused",
    daily_spend_limit_micros=50_000_000,
    targeting_country_codes=["US"],
    description="Initial pilot",
    idempotency_key="hyper-us-test-2026-06-25",
)
```

```text
openai_ads_ad_groups_create(
    campaign_id="CAMPAIGN_ID",
    name="US - Marketing Operators",
    status="paused",
    max_bid_micros=2_000_000,
    context_hints=[
        "user is asking about ad automation",
        "user wants to scale paid acquisition",
    ],
    idempotency_key="hyper-us-test-adgroup-2026-06-25",
)
```

```text
openai_ads_images_upload(image_url="https://cdn.example.com/asset.png")
```

```text
openai_ads_create(
    ad_group_id="AD_GROUP_ID",
    name="US - Marketing - Variant A",
    title="Run ads on autopilot",
    body="Hyper builds, ships, and optimizes ads for you.",
    target_url="https://example.com?utm_source=openai_ads",
    file_id=FILE_ID_FROM_UPLOAD,
    status="paused",
    creative_type="chat_card",
    idempotency_key="hyper-us-test-ad-a-2026-06-25",
)
```

Surface the returned `review_status`.

## Product Feed Flow

For product-feed ads:

- Create the campaign with `mode="product_feed"`.
- Create an ad group with `product_feed_id` and optional `product_set_filters`.
- Create one `product_ad_template` ad in that ad group.

```text
openai_ads_campaigns_create(
    name="Catalog retargeting",
    status="paused",
    daily_spend_limit_micros=100_000_000,
    mode="product_feed",
    bidding_type="clicks",
)
```

```text
openai_ads_ad_groups_create(
    campaign_id="CAMPAIGN_ID",
    name="High intent catalog",
    status="paused",
    max_bid_micros=3_000_000,
    product_feed_id="feed_123",
    product_set_filters=[
        {"field": "availability", "operator": "in", "values": ["in_stock"]},
    ],
)
```

```text
openai_ads_create(
    ad_group_id="AD_GROUP_ID",
    name="Catalog template",
    title="{{product.title}}",
    body="Shop {{product.brand}} today.",
    price="{{product.price}}",
    status="paused",
    creative_type="product_ad_template",
)
```

## Custom Audiences

Use custom audiences for inclusion, exclusion, and bid multipliers.

```text
openai_ads_create_custom_audience(
    name="Newsletter leads",
    members=[
        {"identifier_type": "email_sha256", "value": "HASHED_EMAIL"},
    ],
)
```

Then attach audience IDs to a campaign:

```text
openai_ads_campaigns_create(
    name="Lead retargeting",
    status="paused",
    daily_spend_limit_micros=25_000_000,
    custom_audience_ids=["aud_123"],
    excluded_custom_audience_ids=["aud_456"],
)
```

Use `custom_audience_bid_multipliers` on ad groups when the user explicitly
wants audience-level bid adjustments.

## Conversions

Set up conversion measurement before attaching conversion settings to a
campaign:

```text
openai_ads_create_conversion_pixel(name="Website pixel")

openai_ads_create_conversion_event_setting(
    name="Signup",
    event_type="custom",
    custom_event_name="signup",
    attribution_window_days=30,
    source_ids=["src_123"],
)
```

Attach conversion event setting IDs on campaign create/update:

```text
openai_ads_campaigns_update(
    campaign_id="CAMPAIGN_ID",
    conversion_event_setting_ids=["ces_123"],
)
```

## Activation

After the user reviews the paused tree:

```text
openai_ads_campaigns_activate(campaign_id=CID)
openai_ads_ad_groups_activate(ad_group_id=AGID)
openai_ads_activate(ad_id=AID)
```

Activation only takes effect at the ad level once `review_status="approved"`.
Poll `openai_ads_get(ad_id=...)` if you need to confirm review state.

## Insights

Insights endpoints are server-aggregated. Use current API query names:

- `time_granularity`: `"hourly"`, `"daily"`, `"monthly"`, `"none"`.
- `aggregation_level`: `"ad_account" | "campaign" | "ad_group" | "ad"`.
- `time_ranges`: JSON objects such as `{"type":"date_range","since":"2026-05-01","until":"2026-05-07"}`. The toolkit also accepts legacy `"YYYY-MM-DD..YYYY-MM-DD"` strings and converts them.
- `fields`: dot-style fields, e.g. `["metadata.readable_time", "campaign.id", "campaign.name", "campaign.spend"]`.
- `filters`: JSON objects such as `{"field":"campaign.id","operator":"IN","value":["cmp_xxx"]}`.
- `sort`: JSON objects such as `{"field":"campaign.spend","direction":"desc"}`.
- `segments`: optional `product`, `country`, or `device`.
- `includes`: optional `zero_impression_items` or `zero_impression_products`.

```text
openai_ads_account_insights_get(
    time_granularity="daily",
    aggregation_level="campaign",
    time_ranges=[
        {"type": "date_range", "since": "2026-05-01", "until": "2026-05-07"}
    ],
    fields=[
        "metadata.readable_time",
        "campaign.id",
        "campaign.name",
        "campaign.clicks",
        "campaign.impressions",
        "campaign.spend",
    ],
    sort=[{"field": "campaign.spend", "direction": "desc"}],
)
```

For a single campaign use `openai_ads_campaign_insights_get(campaign_id=...)`;
similar for ad groups and ads.

For conversion totals:

```text
openai_ads_get_conversion_insights(
    aggregation_level="campaign",
    time_ranges=[
        {"type": "date_range", "since": "2026-05-01", "until": "2026-05-07"}
    ],
    entity_ids=["cmp_123"],
)
```

## Status Flow

`paused` <-> `active` -> `archived` (irreversible) at campaign, ad group,
ad, and custom audience level. Account-level tools support activate and pause.
Use the dedicated tools for status changes.

Ads also have:

- `review_status="in_review"`: not delivering yet.
- `review_status="approved"`: eligible to deliver.
- `review_status="rejected"`: needs a new creative.

## Cache Snapshot

The connect-time context builder writes a snapshot of the account into
`integration.toolkit_settings["openai_ads_cache"]`. Use:

- `openai_ads_caches_get()` for cheap read-only access.
- `openai_ads_caches_refresh()` to refresh account, campaigns, ad groups, ads, and insights.

Refresh sparingly: once per session is plenty unless something feels stale.

## Pagination

All list endpoints use cursor pagination:

- `limit` (default 20, max 500).
- `after` / `before`: pass the previous response's `last_id` / `first_id`.
- `order`: `"asc"` | `"desc"`.

Responses include `has_more`. Keep paging only when true and only when you
actually need everything.

## Health Check

After connecting, run `openai_ads_health_check()`.

If `connected=false`, the API key is missing, invalid, or expired. Tell the
user to regenerate it from the Ads Manager Settings API area.

## Safety Rules

**Never:**

- Pass dollar amounts directly. All money inputs are micros.
- Activate a campaign, ad group, ad, or account without explicit user approval.
- Skip image upload for a `chat_card`; it needs a real `file_id`.
- Use geo exclusions; use included geos and audience exclusions instead.
- Assume an ad is delivering just because `status="active"`. Always check `review_status`.
- Treat `archive` as reversible.
- Promise paid traffic on a new ad. New ads sit in `review_status="in_review"` until OpenAI approves them.
