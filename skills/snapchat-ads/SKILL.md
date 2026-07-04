---
name: snapchat-ads
description: Plan and create Snapchat Ads campaigns end-to-end via the Hyper MCP — campaign, ad squad, ad build order with media upload, creatives, targeting, Snap Pixel conversion tracking, and sync / async reporting, using micro-currency budgets and JSON-Patch partial updates. Use when the user wants to launch Snapchat ads, build ad squads, upload Snap creatives, set up Snap Pixel tracking, or analyze Snapchat ad performance. Also triggers on snap ads, snapchat campaign, or snapchat ads manager.
---

# Snapchat Ads Campaigns

Strategic guide for managing Snapchat advertising via the Snapchat Marketing API v1. The toolkit is a raw-HTTP client, so parameter types must be sent exactly as documented — strings as strings, integers as integers, and all money in **micro-currency**.

## Requirements

- **Hyper MCP installed and connected.** [https://app.hyperfx.ai/mcp](https://app.hyperfx.ai/mcp)
- **Snapchat Ads integration connected** (a Snap Business account with ad account access) at [https://app.hyperfx.ai/integrations](https://app.hyperfx.ai/integrations).

If `snapchat_ads_list_organizations` is not in the tool list, stop and tell the user to enable Hyper MCP and connect Snapchat Ads. Creating ad squads or serving ads also requires an **active funding source** on the ad account — campaigns, media, and creatives can be created without one.

## Out of scope — defer to other skills

- **Creative generation** (ad copy, images, video) → [`ad-creative-generation`](../ad-creative-generation) / [`video-generation`](../video-generation).
- **Cross-platform campaign launches** → use this skill for Snapchat, then invoke `meta-ads` / `tiktok-ads` separately.

## Critical Rules

> **CRITICAL**: All budgets and bids are in **micro-currency**. $1.00 = 1,000,000 micros. $50/day = `50000000`. Never pass dollar amounts directly. Minimum campaign daily budget / lifetime spend cap is **$20** (`20000000`).

> **CRITICAL**: Create campaigns, ad squads, and ads with `status="PAUSED"`. Never launch live without explicit user review. Because everything defaults to PAUSED, you can build a full campaign and delete it without spending — ideal for demos.

> **CRITICAL**: Updates are **JSON-Patch partial updates** — pass only the fields you want to change, parented on the PARENT entity. `snapchat_ads_update_campaign` needs `ad_account_id`; `snapchat_ads_update_ad_squad` needs `campaign_id` (NOT ad_account_id); `snapchat_ads_update_ad` needs `ad_squad_id`.

> **CRITICAL**: Account-level stats (`entity_type="adaccounts"`) only support `fields="spend"`. For impressions / swipes / conversions, query at the **campaign, adsquad, ad, creative, or media** level.

> **CRITICAL**: There is **no `time_range` preset** (e.g. `LAST_30_DAYS` does not exist). Pass `start_time` and `end_time` (ISO 8601). Omitting both returns **lifetime** totals, not a recent window.

> **CRITICAL**: Ad squad `billing_event` is always `IMPRESSION`. `bid_micro` is required for `bid_strategy="LOWEST_COST_WITH_MAX_BID"` and `"TARGET_COST"`; omit it for `"AUTO_BID"`.

> **IMPORTANT**: Creating an ad squad requires an active **funding source** on the ad account. If the account is PENDING / unfunded, ad squad creation fails — that is a billing state, not a bug.

> **IMPORTANT**: Creatives require `top_snap_media_id` (an uploaded media id) and a `headline` (≤ 34 chars). `brand_name` is ≤ 32 chars.

## Tool surface

| Group | Tools |
| --- | --- |
| Org & accounts | `snapchat_ads_list_organizations`, `snapchat_ads_list_ad_accounts`, `snapchat_ads_get_ad_account` |
| Campaigns | `snapchat_ads_list_campaigns`, `snapchat_ads_get_campaign`, `snapchat_ads_create_campaign`, `snapchat_ads_update_campaign`, `snapchat_ads_delete_campaign` |
| Ad squads | `snapchat_ads_list_ad_squads`, `snapchat_ads_get_ad_squad`, `snapchat_ads_create_ad_squad`, `snapchat_ads_update_ad_squad`, `snapchat_ads_delete_ad_squad`, `snapchat_ads_estimate_ad_squad_outcomes` |
| Ads | `snapchat_ads_list_ads`, `snapchat_ads_get_ad`, `snapchat_ads_create_ad`, `snapchat_ads_update_ad`, `snapchat_ads_delete_ad` |
| Creatives | `snapchat_ads_list_creatives`, `snapchat_ads_get_creative`, `snapchat_ads_create_creative` |
| Media | `snapchat_ads_list_media`, `snapchat_ads_create_media`, `snapchat_ads_upload_media` |
| Reporting | `snapchat_ads_get_stats`, `snapchat_ads_create_stats_report`, `snapchat_ads_get_stats_report` |
| Conversion | `snapchat_ads_list_pixels`, `snapchat_ads_list_custom_conversions` |
| Targeting | `snapchat_ads_search_targeting` |

## Phase 1: Org & Account Discovery

Snapchat nests ad accounts under organizations. Start here:

```python
snapchat_ads_list_organizations(with_ad_accounts=True)
```

- If multiple organizations / accounts: ask the user to select one.
- If single: inform the user and proceed.
- Note the `ad_account_id` (and `organization_id`) — `ad_account_id` is required for most calls.

List accounts for a specific org explicitly with `snapchat_ads_list_ad_accounts(organization_id="<ORG_ID>")`.

## Phase 2: Account Assessment

Run in parallel to understand the account state:

```python
snapchat_ads_list_campaigns(ad_account_id="<AD_ACCOUNT_ID>")
snapchat_ads_list_ad_squads(ad_account_id="<AD_ACCOUNT_ID>")
snapchat_ads_list_ads(ad_account_id="<AD_ACCOUNT_ID>")
snapchat_ads_list_media(ad_account_id="<AD_ACCOUNT_ID>")
snapchat_ads_list_pixels(ad_account_id="<AD_ACCOUNT_ID>")
```

Then confirm: objective, daily/lifetime budget (convert dollars → micros), audience (geo/age/gender/interests/devices), creative assets + destination URL, and — for conversion goals — that a pixel/custom conversion exists.

## Phase 3: Campaign Structure

### Hierarchy

```
Organization
└── Ad Account ── Creatives ── Media
    └── Campaign (objective, optional budget)
        └── Ad Squad (targeting, bid, budget, optimization_goal, placement)
            └── Ad (links a Creative)
```

### Build order

1. **Campaign** (`status="PAUSED"`)
2. **Media**: `create_media` (container) → `upload_media` (file)
3. **Creative**: `create_creative` using the uploaded `top_snap_media_id`
4. **Ad squad** under the campaign (requires funding source)
5. **Ad** under the ad squad, linking the `creative_id`

Assets (2–3) and the ad squad (4) are independent; the ad (5) needs both a `creative_id` and an `ad_squad_id`.

### Campaign objectives

Pass the classic `objective` string, or newer objectives via `extra={"objective_v2_properties": {"objective_v2_type": "..."}}`.

| `objective` | Use case |
| --- | --- |
| `BRAND_AWARENESS` | Reach / impressions |
| `ENGAGEMENT` | Engagement, story opens |
| `VIDEO_VIEW` | Video views |
| `WEB_VIEW` | Traffic |
| `WEB_CONVERSION` | Purchases, signups, leads (pixel) |
| `APP_INSTALL` / `APP_REENGAGEMENT` | App installs / re-engagement |
| `LEAD_GENERATION` | On-Snap lead forms |
| `CATALOG_SALES` | Dynamic product ads |

`objective_v2_type` values: `AWARENESS_AND_ENGAGEMENT`, `TRAFFIC`, `SALES`, `APP_PROMOTION`, `LEADS`.

## Phase 4: Campaign Creation

### 1. Create campaign

```python
snapchat_ads_create_campaign(
    ad_account_id="<AD_ACCOUNT_ID>",
    name="Spring Launch 2026",
    objective="WEB_CONVERSION",
    status="PAUSED",
    start_time="2026-07-01T00:00:00-07:00",
    daily_budget_micro=50000000,   # $50.00/day, min $20
)
```

`buy_model` is `AUCTION` (default) or `RESERVED`. Use `daily_budget_micro` or `lifetime_spend_cap_micro`.

### 2. Upload media (two steps)

> **MEDIA SPECS — wrong specs cause creative error E2601** ("media cannot be used to create any creative type"). Top Snap image/video must be **1080×1920 px (9:16)**. Images: PNG/JPG, ≤ 5 MB. Video: MP4/MOV, 3–180 s. **If you generate the asset, generate it at 1080×1920.** Snapchat tags valid media with `media_usages` (e.g. `TOP_SNAP`); media with no usable usage is rejected at the creative step.

```python
snapchat_ads_create_media(
    ad_account_id="<AD_ACCOUNT_ID>",
    name="Spring Hero Video",
    type="VIDEO",            # IMAGE | VIDEO | LENS_PACKAGE | PLAYABLE — must match the file
)
# PREFER file_id (a Hyper file id, e.g. a generated image/video). It is read
# straight from storage, so it works even when a public URL isn't reachable
# from the backend. Use file_url / file_path only for genuinely external assets.
snapchat_ads_upload_media(
    media_id="<MEDIA_ID>",
    file_id="<HYPER_FILE_ID>",   # or file_url="https://..." / file_path="/server/path"
)
```

`upload_media` auto-chunks files larger than 32 MB (up to 1 GB). Confirm the returned `media_status` is `READY` before creating a creative.

### 3. Create creative

```python
snapchat_ads_create_creative(
    ad_account_id="<AD_ACCOUNT_ID>",
    name="Spring Hero Creative",
    type="WEB_VIEW",                  # SNAP_AD | WEB_VIEW | APP_INSTALL | DEEP_LINK | ...
    top_snap_media_id="<MEDIA_ID>",
    headline="Shop the Spring Drop",  # <= 34 chars (required)
    brand_name="Acme",                # <= 32 chars
    call_to_action="SHOP_NOW",
    web_view_properties={"url": "https://example.com/spring"},
)
```

> **`profile_properties` is required for ALL creative types on most accounts** (not just SNAP_AD): `profile_properties={"profile_id": "<PUBLIC_PROFILE_ID>"}`. If you don't have the public profile id, call `snapchat_ads_list_creatives` and copy `profile_properties.profile_id` from any existing creative. Without it, creation fails — previously silently, now with a clear error.

`call_to_action` is type-dependent (WEB_VIEW → `VIEW`/`SHOP_NOW`/`SIGN_UP`; APP_INSTALL → `INSTALL_NOW`/`DOWNLOAD`; DEEP_LINK → `OPEN_APP`/`PLAY`). WEB_VIEW needs `web_view_properties={"url": ...}`; APP_INSTALL/DEEP_LINK use `app_install_properties`/`deep_link_properties`.

### 4. Create ad squad

> **CRITICAL**: Required — `campaign_id`, `name`, `optimization_goal`, `targeting`, `billing_event="IMPRESSION"`, `bid_strategy`, a budget (`daily_budget_micro` OR `lifetime_budget_micro`), and `placement_v2`. `bid_micro` is required unless `bid_strategy="AUTO_BID"`.

```python
snapchat_ads_create_ad_squad(
    campaign_id="<CAMPAIGN_ID>",
    name="US Gen-Z 18-24",
    status="PAUSED",
    type="SNAP_ADS",                     # SNAP_ADS | LENS | FILTER
    optimization_goal="PIXEL_PURCHASE",
    billing_event="IMPRESSION",
    bid_strategy="LOWEST_COST_WITH_MAX_BID",
    bid_micro=3000000,                   # required for non-AUTO_BID
    daily_budget_micro=20000000,         # ad-squad min is $5 (5000000)
    placement_v2={"config": "AUTOMATIC"},
    # pixel_id="<PIXEL_ID>",             # required-ish for PIXEL_* goals
    targeting={
        "geos": [{"country_code": "us"}],
        "demographics": [{"min_age": 18, "max_age": 24}],
        "regulated_content": False,
    },
)
```

**`optimization_goal`:** `IMPRESSIONS`, `SWIPES`, `VIDEO_VIEWS`, `VIDEO_VIEWS_15_SEC`, `USES`, `STORY_OPENS`, `APP_INSTALLS`, `LANDING_PAGE_VIEW`, `LEAD_FORM_SUBMISSIONS`, `PIXEL_PAGE_VIEW`, `PIXEL_ADD_TO_CART`, `PIXEL_PURCHASE`, `PIXEL_SIGNUP`, `APP_ADD_TO_CART`, `APP_PURCHASE`, `APP_SIGNUP`, `APP_REENGAGE_OPEN`, `APP_REENGAGE_PURCHASE`.

**`bid_strategy`:** `AUTO_BID` (no `bid_micro`), `LOWEST_COST_WITH_MAX_BID` (`bid_micro` = max bid), `TARGET_COST` (`bid_micro` = target). `MIN_ROAS` is deprecated.

**`pacing_type`:** `STANDARD` (default) or `ACCELERATED`. **`placement_v2`:** `{"config": "AUTOMATIC"}` or `{"config": "CUSTOM", ...}`.

#### Targeting object

```python
targeting = {
    "regulated_content": False,
    "geos": [{"country_code": "us"}],                 # lowercase ISO country
    "demographics": [{"min_age": 18, "max_age": 34, "gender": "FEMALE"}],
    "interests": [{"category_id": ["<SCLS_ID>"]}],    # from snapchat_ads_search_targeting
    "devices": [{"os_type": "iOS"}],
}
```

#### Optional: forecast before launch

`snapchat_ads_estimate_ad_squad_outcomes` returns daily/weekly reach, conversion, and impression ranges for a proposed squad config — without creating anything. Useful to sanity-check budget/targeting before building:

```python
snapchat_ads_estimate_ad_squad_outcomes(
    ad_account_id="<AD_ACCOUNT_ID>",
    ad_squad={
        "optimization_goal": "IMPRESSIONS",
        "bid_strategy": "AUTO_BID",
        "daily_budget_micro": 50000000,
        "type": "SNAP_ADS",
        "placement_v2": {"config": "AUTOMATIC"},
        "targeting": {"geos": [{"country_code": "us"}]},
        "start_time": "2026-07-01T00:00:00-07:00",
    },
)
```

The tool auto-adds `delivery_constraint`. If it returns OE101/`UNSUPPORTED_DELIVERY_CONSTRAINT`, outcome estimation is likely **unavailable for this account type** (PARTNER accounts are often excluded) — skip forecasting and use the Snapchat Ads Manager reach planner instead.

### 5. Create ad

> **Omit `type`** — it's auto-derived from the creative (a WEB_VIEW creative needs a `REMOTE_WEBPAGE` ad, SNAP_AD→SNAP_AD, APP_INSTALL→APP_INSTALL, DEEP_LINK→DEEP_LINK). Only set `type` to override.

```python
snapchat_ads_create_ad(
    ad_squad_id="<AD_SQUAD_ID>",
    name="Spring Hero Ad",
    creative_id="<CREATIVE_ID>",
    status="PAUSED",
)
```

## Phase 5: Conversion Tracking

For `PIXEL_*` / `APP_*` goals, discover the pixel and its custom conversions, then use the conversion id as a stats metric:

```python
snapchat_ads_list_pixels(ad_account_id="<AD_ACCOUNT_ID>")
snapchat_ads_list_custom_conversions(pixel_id="<PIXEL_ID>")     # or mobile_app_id="<APP_ID>"
```

Each custom conversion `id` is prefixed `conversion_` and can be passed in the `fields` of a stats call (e.g. `conversion_0029904941`).

## Phase 6: Targeting Lookups

`snapchat_ads_search_targeting(path=...)` — `path` is appended to `/targeting/`:

| What | `path` | Notes |
| --- | --- | --- |
| Countries | `geo/country` | |
| Regions / metros | `geo/{country_code}/region`, `geo/{country_code}/metro` | nested |
| Age / gender / languages | `demographics/age_group`, `demographics/gender`, `demographics/languages` | |
| Interests (Snap Lifestyle Categories) | `interests/scls` | **requires** `extra={"country_code": "us"}` |
| Interests (other taxonomies) | `interests/dlxs`, `interests/nln` | |
| Devices | `device/os_type`, `device/carrier`, `device/marketing_name` | |

```python
snapchat_ads_search_targeting(path="interests/scls", extra={"country_code": "us"})
```

## Phase 7: Reporting

### Synchronous stats

```python
# Account level — SPEND ONLY
snapchat_ads_get_stats(
    entity_type="adaccounts", entity_id="<AD_ACCOUNT_ID>",
    fields="spend", granularity="DAY",
    start_time="2026-06-01T00:00:00-07:00", end_time="2026-06-30T00:00:00-07:00",
)

# Campaign / ad squad / ad — full metrics
snapchat_ads_get_stats(
    entity_type="campaigns", entity_id="<CAMPAIGN_ID>",
    fields="impressions,swipes,spend,video_views,conversion_purchases",
    granularity="DAY",
    start_time="2026-06-01", end_time="2026-06-30",
    ad_account_id="<AD_ACCOUNT_ID>",   # lets the tool align DAY/HOUR to the account timezone
    breakdown="adsquad",
)
```

- **`spend` is micro-currency** — divide by 1,000,000 for dollars.
- `granularity`: `TOTAL`, `DAY`, `HOUR`, `LIFETIME`. For **DAY/HOUR**, pass `ad_account_id` (or `account_timezone`) and the tool auto-aligns `start_time`/`end_time` to the account-timezone midnight / hour boundaries Snapchat requires — no manual UTC math. At the adaccounts level the timezone is resolved automatically.
- `breakdown`: `campaign`/`adsquad`/`ad`. `report_dimension`: `country`, `region`, `dma`, `age`, `gender`, `os`, `make`, `lifestyle_category`.
- Stats finalize ~48h after the day ends in the account timezone — recent numbers can be partial.

### Asynchronous reports (large pulls / CSV export / reach overlap)

```python
snapchat_ads_create_stats_report(
    entity_type="campaigns", entity_id="<CAMPAIGN_ID>",
    fields="impressions,spend", granularity="DAY",
    start_time="2026-01-01T00:00:00-08:00", end_time="2026-06-30T00:00:00-07:00",
    async_format="csv",
)
# poll until report_run_status == COMPLETED, then read .result (signed download URL)
snapchat_ads_get_stats_report(
    entity_type="campaigns", entity_id="<CAMPAIGN_ID>", report_run_id="<REPORT_RUN_ID>",
)
```

For reach & frequency overlap (same ad account only): `overlap=True`, `overlap_type="campaign"`, `ids="<id1>,<id2>"`.

## Update & Delete

Updates are partial (JSON Patch) — send only what changes, with the correct parent id:

```python
snapchat_ads_update_campaign(ad_account_id="<AD_ACCOUNT_ID>", campaign_id="<ID>", status="ACTIVE")
snapchat_ads_update_ad_squad(campaign_id="<CAMPAIGN_ID>", ad_squad_id="<ID>", daily_budget_micro=30000000)
snapchat_ads_update_ad(ad_squad_id="<AD_SQUAD_ID>", ad_id="<ID>", status="PAUSED")
```

Deletes are permanent: `snapchat_ads_delete_campaign`, `snapchat_ads_delete_ad_squad`, `snapchat_ads_delete_ad`.

## Campaign Workflow

Discover org/account → audit account → research (objective, audience, budget, assets) → confirm strategy → create campaign (`PAUSED`) → upload media → create creative → create ad squad → create ad → review with user → activate (`update_*` to `ACTIVE`).

## Known Limitations

| Issue | Workaround |
| --- | --- |
| Creative creation fails (historically a silent null error) | Almost always a missing `profile_properties.profile_id` (required for ALL creative types) — pull one from `list_creatives`. The tool now raises a clear error instead of a null "success". |
| Ad creation fails with a type mismatch | Omit `type` so it's derived from the creative (WEB_VIEW → REMOTE_WEBPAGE, etc.). |
| Forecast returns OE101 / UNSUPPORTED_DELIVERY_CONSTRAINT | The tool auto-adds `delivery_constraint`; if it persists, outcome estimation is unavailable for the account type (often PARTNER) — use the Ads Manager reach planner. |
| DAY-granularity stats reject the time range (E1008) | Handled automatically — pass `ad_account_id` to `get_stats`/`create_stats_report` and the tool aligns to account-timezone midnight + hour boundaries. |
| Media rejected with E2601 ("cannot be used to create any creative type") | The file doesn't meet specs — use a 1080×1920 (9:16) PNG/JPG ≤ 5 MB image or 1080×1920 MP4/MOV. Re-generate the asset at the right size, re-upload, and confirm `media_status` is `READY`. |
| Upload fails on file_path / internal URL (Errno 2 / DNS error) | Pass `file_id` (the Hyper file id) instead — it reads from storage directly. Avoid server-local paths and internal `files.*` URLs. |
| Account-level stats reject non-`spend` fields (error E1008) | Query other metrics at campaign/adsquad/ad/creative/media level. |
| No `time_range`/date preset | Pass `start_time` + `end_time`; omitting them returns lifetime totals. |
| Ad squad creation fails on unfunded accounts | Add a funding source; campaigns/media/creatives still work without one. |
| `bid_micro` required for non-AUTO_BID strategies | Provide `bid_micro` for `LOWEST_COST_WITH_MAX_BID` / `TARGET_COST`. |
| Media > 32 MB | Handled automatically (chunked upload, up to 1 GB). |
| Partial updates | Use the update tools (PATCH) with the correct parent id — never resend the whole object. |

## Safety Rules

**Never:**

- Assume organization, ad account, campaign, media, or creative IDs — look them up or ask.
- Skip the account audit phase.
- Create campaigns, ad squads, or ads without explicit user approval.
- Set anything to `ACTIVE` without user consent.
- Pass dollar amounts instead of micro-currency.
- Query non-`spend` metrics at the `adaccounts` level.
- Use a `time_range` preset — always pass `start_time`/`end_time`.
- Use `LOWEST_COST_WITH_MAX_BID` / `TARGET_COST` without `bid_micro`.
- Pass the wrong parent id to update tools (ad squad → `campaign_id`, ad → `ad_squad_id`).
