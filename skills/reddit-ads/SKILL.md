---
name: reddit-ads
description: Plan and create Reddit Ads campaigns end-to-end via the Hyper MCP — campaign, ad group, ad build order with subreddit / interest / geo targeting, promoted posts, custom and saved audiences, Reddit pixel conversion tracking, and reporting, with micro-currency budgets. Use when the user wants to launch Reddit ads, promote a Reddit post, target subreddits, set up a Reddit pixel, or pull Reddit ad reports. Also triggers on reddit campaign, reddit ppc, or reddit ads manager.
---

# Reddit Ads Campaigns

Strategic guide for managing Reddit advertising via the Reddit Ads API v3. The toolkit is a raw-HTTP client, so parameter types must be sent exactly as documented — strings as strings, integers as integers, and all money in **micro-currency**.

## Requirements

- **Hyper MCP installed and connected.** [https://app.hyperfx.ai/mcp](https://app.hyperfx.ai/mcp)
- **Reddit Ads integration connected** (a Reddit Ads business account with ad account access) at [https://app.hyperfx.ai/integrations](https://app.hyperfx.ai/integrations).

If `reddit_ads_get_me` / `reddit_ads_list_businesses` is not in the tool list, stop and tell the user to enable Hyper MCP and connect Reddit Ads. Serving ads also requires an active **funding instrument** on the ad account — campaigns and ad groups can be created without one, but the account must be funded for ads to run.

## Critical Rules

> **CRITICAL**: All budgets and bids are in **micro-currency**. $1.00 = 1,000,000 micros. A $50/day budget is `goal_value=50000000`; a $0.50 CPC is `bid_value=500000`. Never pass dollar amounts directly.

> **CRITICAL**: Create campaigns, ad groups, and ads with `configured_status="PAUSED"` (the toolkit default). Never launch live without explicit user review. Because everything defaults to PAUSED, you can build a full campaign and archive it without spending — ideal for demos.

> **CRITICAL**: Discovery flows through **businesses**, not a flat account list: `reddit_ads_get_me` → `reddit_ads_list_businesses` → `reddit_ads_list_ad_accounts(business_id=...)`. There is no top-level "list all ad accounts" call.

> **CRITICAL**: An **ad promotes a post** — it needs a `post_id` (or a `profile_id` + creation), not raw creative fields. Before creating an ad, find an existing post (`reddit_ads_list_posts`) or create one (`reddit_ads_create_post`). Only **community posts** can be promoted via the API.

> **CRITICAL**: The ad group `targeting.communities` array takes **subreddit NAMES, not the `t5_` IDs** that `search_targeting` returns. Use the `name` field from the search results: `"communities": ["marketing", "SaaS", "Entrepreneur"]` — passing `t5_...` IDs fails with "You cannot set invalid communities". (`interests` does use the IDs; `geolocations` uses country codes like `"US"`.)

> **CRITICAL**: Resolve targeting with `reddit_ads_search_targeting` before building — never guess values. `targeting` is a named parameter on `create_ad_group`; pass any other variant-heavy fields (audiences, schedule) via `input_data`.

> **CRITICAL**: `start_time` is **required when creating an ad group** (ISO 8601). On a **campaign**, `start_time` is only valid with CBO enabled — for non-CBO campaigns, omit it at the campaign level and set it on the ad group, or you get "Cannot set start_time for a campaign without having campaign budget optimization enabled."

> **CRITICAL**: `bid_value` is **always required** for `bid_type` CPC/CPM/CPV/CPV6 — even with `bid_strategy="MAXIMIZE_VOLUME"`. Set a cap (e.g. `bid_value=600000` = $0.60) or get one from `reddit_ads_generate_bid_suggestion`.

> **CRITICAL**: There is **no instant hard delete**. Reddit requires an entity to be **ARCHIVED for 3+ hours** before it can be permanently DELETED. `reddit_ads_delete_*` tries the real delete and **falls back to archiving** (which stops delivery) when it isn't deletable yet. Check the returned `configured_status` (ARCHIVED vs DELETED) and tell the user a hard delete is possible ~3h after archiving.

> **CRITICAL**: Reporting (`reddit_ads_get_report`) requires `fields`, `starts_at`, and `ends_at`. Spend metrics come back in **micro-currency**.

> **IMPORTANT**: Campaigns require `name`, `objective`, and (to serve) a `funding_instrument_id` — list them with `reddit_ads_list_funding_instruments`. Conversion-optimized ad groups need a `conversion_pixel_id` (`reddit_ads_list_pixels`). From **2026-07-13**, ad groups and CBO campaigns require `conversion_pixel_id`.

> **IMPORTANT**: Lead generation works as a campaign **objective** (`LEAD_GENERATION`), but lead-gen **form management is not available** via this toolkit (it needs a feature-gated scope) — build lead-gen forms in Reddit Ads Manager.

> **CRITICAL** (CONVERSIONS campaigns): the ad group `optimization_goal` is **required** and, for **standard (non-CBO) CONVERSIONS campaigns, the only accepted value is `CLICKS`**. `SIGN_UP`, `PAGE_VISIT`, `PURCHASE`, etc. are valid enum values but are **rejected for this campaign type** ("Conversion goal ... is not valid for Conversions Campaigns"). Reddit validates in two layers — enum membership, then campaign-type allowance. Use `optimization_goal="CLICKS"`; the CONVERSIONS objective still governs delivery.

> **CRITICAL** (post types): valid post `type` values are **`IMAGE`, `VIDEO`, `CAROUSEL`, `TEXT`** — **not `LINK`**. **`TEXT` posts are "free-form ads": they cannot carry a `click_url` and CANNOT be used in CONVERSIONS-campaign ads.** For any ad with a destination (especially CONVERSIONS), promote an **IMAGE / VIDEO / CAROUSEL** post.

> **CRITICAL** (IMAGE/VIDEO posts): the media + CTA go in a **`content[]`** block (via `input_data`), not as top-level fields. An IMAGE post needs `content[].media_url` pointing to an **already-uploaded Reddit asset** — discover one with `reddit_ads_list_creative_assets(profile_id=...)`. `call_to_action` lives **inside `content[]`** and uses **display-label strings** (`"Sign Up"`, `"Learn More"`, `"Shop Now"`), NOT enum constants (`"SIGN_UP"`). TEXT posts must have an **empty** `content[]`.

## Tool surface

| Group | Tools |
| --- | --- |
| Identity & discovery | `reddit_ads_get_me`, `reddit_ads_list_businesses`, `reddit_ads_list_ad_accounts`, `reddit_ads_get_ad_account`, `reddit_ads_list_funding_instruments` |
| Campaigns | `reddit_ads_list_campaigns`, `reddit_ads_get_campaign`, `reddit_ads_create_campaign`, `reddit_ads_update_campaign`, `reddit_ads_delete_campaign` |
| Ad groups | `reddit_ads_list_ad_groups`, `reddit_ads_get_ad_group`, `reddit_ads_create_ad_group`, `reddit_ads_update_ad_group`, `reddit_ads_delete_ad_group` |
| Ads | `reddit_ads_list_ads`, `reddit_ads_get_ad`, `reddit_ads_create_ad`, `reddit_ads_update_ad`, `reddit_ads_delete_ad` |
| Posts & creative | `reddit_ads_list_profiles`, `reddit_ads_get_profile`, `reddit_ads_list_posts`, `reddit_ads_create_post`, `reddit_ads_get_post`, `reddit_ads_list_creative_assets`, `reddit_ads_get_creative_asset` |
| Targeting | `reddit_ads_search_targeting`, `reddit_ads_suggest_keywords`, `reddit_ads_validate_keywords`, `reddit_ads_validate_geolocations` |
| Audiences | `reddit_ads_list_custom_audiences`, `reddit_ads_create_custom_audience`, `reddit_ads_get_custom_audience`, `reddit_ads_delete_custom_audience`, `reddit_ads_update_custom_audience_users`, `reddit_ads_list_saved_audiences`, `reddit_ads_create_saved_audience`, `reddit_ads_get_saved_audience`, `reddit_ads_update_saved_audience` |
| Pixels & Conversions API | `reddit_ads_list_pixels`, `reddit_ads_list_pixels_by_business`, `reddit_ads_post_conversion_events`, `reddit_ads_get_pixel_last_fired_at` |
| Forecasting & access | `reddit_ads_generate_bid_suggestion`, `reddit_ads_get_feature_access` |
| Apps / SKAdNetwork | `reddit_ads_list_apps`, `reddit_ads_get_skan_availability` |
| Reporting | `reddit_ads_get_report` |

## Phase 1: Business & Account Discovery

Reddit nests ad accounts under businesses. Start here:

```python
reddit_ads_get_me()
reddit_ads_list_businesses()
```

- For each business, list its ad accounts: `reddit_ads_list_ad_accounts(business_id="<BUSINESS_ID>")`.
- If multiple businesses / accounts: ask the user to select one.
- If single: inform the user and proceed.
- Note the `ad_account_id` (often prefixed `t2_` or `a2_`) — it's required for most calls.

## Phase 2: Account Assessment

Run in parallel to understand the account state:

```python
reddit_ads_list_campaigns(ad_account_id="<AD_ACCOUNT_ID>")
reddit_ads_list_ad_groups(ad_account_id="<AD_ACCOUNT_ID>")
reddit_ads_list_ads(ad_account_id="<AD_ACCOUNT_ID>")
reddit_ads_list_funding_instruments(ad_account_id="<AD_ACCOUNT_ID>")
reddit_ads_list_pixels(ad_account_id="<AD_ACCOUNT_ID>")
reddit_ads_list_profiles(ad_account_id="<AD_ACCOUNT_ID>")
```

Then confirm: objective, daily/lifetime budget (convert dollars → micros), targeting (communities/interests/geo/devices), the post to promote, and — for conversion goals — that a pixel exists. Optionally call `reddit_ads_get_feature_access(ad_account_id=...)` to confirm which capabilities (e.g. CBO) the account supports.

## Phase 3: Campaign Structure

### Hierarchy

```
Business
└── Ad Account ── Funding Instruments ── Pixels ── Profiles (── Posts, Creative Assets)
    └── Campaign (objective, funding, optional CBO budget)
        └── Ad Group (targeting, bid, budget, optimization, conversion_pixel_id)
            └── Ad (promotes a Post via post_id)
```

### Build order

1. **Resolve targeting IDs** with `reddit_ads_search_targeting` (subreddits, interests, geo, etc.)
2. **Campaign** (`configured_status="PAUSED"`, objective, `funding_instrument_id`)
3. **Ad group** under the campaign (targeting, bid, budget; `conversion_pixel_id` for conversion goals)
4. **Post**: find one via `reddit_ads_list_posts`, or create one with `reddit_ads_create_post`
5. **Ad** under the ad group, linking the `post_id`

### Campaign objectives

| `objective` | Use case |
| --- | --- |
| `CLICKS` | Traffic to a destination URL |
| `CONVERSIONS` | Purchases / signups / leads (requires a pixel) |
| `IMPRESSIONS` | Reach / awareness |
| `VIDEO_VIEWABLE_IMPRESSIONS` | Video views |
| `APP_INSTALLS` | App installs (requires an app) |
| `CATALOG_SALES` | Dynamic product ads (requires a catalog) |
| `LEAD_GENERATION` | On-Reddit lead forms (build the form in Ads Manager) |

## Phase 4: Campaign Creation

### 1. Create campaign

```python
reddit_ads_create_campaign(
    ad_account_id="<AD_ACCOUNT_ID>",
    name="Spring Launch 2026",
    objective="CLICKS",
    configured_status="PAUSED",
    funding_instrument_id="<FUNDING_INSTRUMENT_ID>",
    # NOTE: do NOT pass start_time here on a non-CBO campaign — it's rejected.
    # Set start_time on the ad group instead (it's CBO-only at the campaign level).
)
```

**Campaign Budget Optimization (CBO):** set `is_campaign_budget_optimization=True` with `goal_type` (`DAILY_SPEND` or `LIFETIME_SPEND`) and `goal_value` (micro-currency). With CBO, the budget lives on the campaign; without it, set the budget on each ad group.

```python
reddit_ads_create_campaign(
    ad_account_id="<AD_ACCOUNT_ID>", name="CBO Test", objective="CONVERSIONS",
    configured_status="PAUSED", funding_instrument_id="<FUNDING_INSTRUMENT_ID>",
    is_campaign_budget_optimization=True, goal_type="DAILY_SPEND", goal_value=50000000,  # $50/day
    conversion_pixel_id="<PIXEL_ID>",
)
```

### 2. Create ad group

Resolve targeting IDs first, then pass the `targeting` block. `bid_type` is `CPC`/`CPM`/`CPV`/`CPV6`; `bid_strategy` is `BIDLESS`/`MANUAL_BIDDING`/`MAXIMIZE_VOLUME`/`TARGET_CPX`. For non-CBO campaigns, set `goal_type` + `goal_value` (the ad group's budget).

```python
reddit_ads_create_ad_group(
    ad_account_id="<AD_ACCOUNT_ID>",
    campaign_id="<CAMPAIGN_ID>",
    name="US — marketing/SaaS subreddits",
    configured_status="PAUSED",
    bid_type="CPC",
    bid_strategy="MANUAL_BIDDING",
    bid_value=600000,                    # $0.60 CPC — REQUIRED for CPC/CPM/CPV/CPV6 (even with MAXIMIZE_VOLUME)
    goal_type="DAILY_SPEND",
    goal_value=20000000,                 # $20/day (omit if the campaign uses CBO)
    start_time="2026-07-01T00:00:00Z",   # REQUIRED on ad groups (ISO 8601)
    conversion_pixel_id="<PIXEL_ID>",    # required for CONVERSIONS / from 2026-07-13
    targeting={
        "geolocations": ["US"],                                # country codes; validate with validate_geolocations
        "communities": ["marketing", "SaaS", "Entrepreneur"],  # subreddit NAMES, NOT the t5_ IDs from search
        "interests": ["<INTEREST_ID>"],                        # interest IDs from search_targeting dimension=interests
    },
)
```

> Pass anything not exposed as a named parameter (extra targeting facets, schedule, etc.) via `input_data` — it's merged into the request body. `start_time` *is* a named param here, but it's easy to miss that it's **required**.

> **For a CONVERSIONS campaign**, the ad group also needs `optimization_goal="CLICKS"` (the only value accepted for non-CBO CONVERSIONS — see the critical rule above) and a `conversion_pixel_id`. Everything else is the same.

#### Optional: bid suggestion before launch

```python
reddit_ads_generate_bid_suggestion(
    ad_account_id="<AD_ACCOUNT_ID>",
    campaign_objective="CLICKS",
    bid_type="CPC",
    # duration is an ISO 8601 start/end range — NOT {"days": N}
    duration={"start_time": "2026-07-01T00:00:00Z", "end_time": "2026-07-08T00:00:00Z"},
    targeting={"geolocations": ["US"]},
)
```

### 3. Find or create a post to promote

`reddit_ads_list_profiles(ad_account_id=...)` finds the profiles whose posts you can promote. Reuse an existing post when possible:

```python
reddit_ads_list_posts(profile_id="<PROFILE_ID>")
```

**Post types:** `IMAGE`, `VIDEO`, `CAROUSEL`, `TEXT` (NOT `LINK`). For an ad with a destination — and for **any CONVERSIONS campaign** — you must use `IMAGE`/`VIDEO`/`CAROUSEL`; **`TEXT` posts are free-form and rejected in CONVERSIONS ads**.

**IMAGE post (the common case).** The media + CTA go in a `content[]` block, and `media_url` must reference an **already-uploaded Reddit asset**. Find one first:

```python
reddit_ads_list_creative_assets(profile_id="<PROFILE_ID>")   # grab media.permanent_url

reddit_ads_create_post(
    profile_id="<PROFILE_ID>",
    type="IMAGE",
    headline="Stop juggling 30 marketing tools — Hyper AI does it all.",
    input_data={
        "content": [{
            "media_url": "https://i.redd.it/<asset>.jpeg",  # from list_creative_assets (or upload one)
            "destination_url": "https://example.com",
            "display_url": "example.com",
            "call_to_action": "Sign Up",                     # display label, NOT "SIGN_UP"
        }],
    },
)
```

> A **TEXT** post takes only `type` + `headline` + (via `input_data`) `post_url`, with an **empty** `content[]` — and again, can't back a CONVERSIONS ad. Use it only for awareness/engagement-style campaigns.

### 4. Create ad

```python
reddit_ads_create_ad(
    ad_account_id="<AD_ACCOUNT_ID>",
    ad_group_id="<AD_GROUP_ID>",
    name="Spring Hero Ad",
    configured_status="PAUSED",
    post_id="<POST_ID>",
    click_url="https://example.com/spring",
)
```

## Phase 5: Audiences

- **Custom audiences** — customer lists, retargeting, lookalikes.
- **Saved audiences** — reusable targeting definitions.

```python
# Customer list: create, then upload hashed members
reddit_ads_create_custom_audience(ad_account_id="<AD_ACCOUNT_ID>", name="CRM list", type="CUSTOMER_LIST")
reddit_ads_update_custom_audience_users(
    audience_id="<AUDIENCE_ID>",
    action_type="ADD",                 # ADD or REMOVE
    column_order=["EMAIL"],
    user_data=[["<sha256_hashed_email>"], ...],   # hash values as Reddit requires
)

# Saved audience from a targeting block
reddit_ads_create_saved_audience(
    ad_account_id="<AD_ACCOUNT_ID>", name="Fitness US", type="<TYPE>",
    targeting={"communities": ["<COMMUNITY_ID>"]},
)
```

Reference a saved/custom audience in an ad group's `targeting` (e.g. via `saved_audience_id` or the audience facet in `input_data`).

## Phase 6: Conversion Tracking (Pixel + Conversions API)

For `CONVERSIONS` goals, discover the pixel and (optionally) send server-side events:

```python
reddit_ads_list_pixels(ad_account_id="<AD_ACCOUNT_ID>")     # get conversion_pixel_id
reddit_ads_get_pixel_last_fired_at(pixel_id="<PIXEL_ID>")   # verify tracking is live

# Conversions API — send server-side events
reddit_ads_post_conversion_events(
    pixel_id="<PIXEL_ID>",
    events=[{ ... }],          # event objects per Reddit's CAPI schema (type, timestamp, hashed user data)
    test_id="<TEST_ID>",       # optional: validate without counting
)
```

## Phase 7: Targeting Lookups

`reddit_ads_search_targeting(dimension=...)` — pass only the params relevant to the dimension:

| What | `dimension` | Params |
| --- | --- | --- |
| Search subreddits | `communities/search` | `query` |
| Subreddits by name | `communities` | `names` |
| Subreddit suggestions | `communities/suggestions` | `names` or `website_url` |
| Interests | `interests` | — |
| Geolocations | `geolocations` | `country` / `cities_search` / `postal_code` |
| Devices / languages / carriers | `devices` / `languages` / `carriers` | — |
| 3rd-party audiences | `third_party_audiences` | — |
| Time zones / industries | `time_zones` / `industries` | — |

```python
reddit_ads_search_targeting(dimension="communities/search", query="fitness")
reddit_ads_search_targeting(dimension="geolocations", country="US")
```

Validate before building: `reddit_ads_suggest_keywords(seed_keywords=[...])`, `reddit_ads_validate_keywords(keywords=[...])`, `reddit_ads_validate_geolocations(geolocation_ids=[...])`.

## Phase 8: Reporting

```python
reddit_ads_get_report(
    ad_account_id="<AD_ACCOUNT_ID>",
    fields=["spend", "impressions", "clicks", "conversions"],
    starts_at="2026-06-01",
    ends_at="2026-06-30",
    breakdowns=["campaign_id"],          # e.g. campaign_id / ad_group_id / date
    time_zone_id="America/Los_Angeles",  # optional
)
```

- **`spend` is micro-currency** — divide by 1,000,000 for dollars.
- `fields`, `starts_at`, and `ends_at` are required; `breakdowns` group the rows.
- Pass additional report request fields via `input_data`.

## Update & Delete

Updates are partial — send only the fields that change:

```python
reddit_ads_update_campaign(campaign_id="<ID>", configured_status="ACTIVE")
reddit_ads_update_ad_group(ad_group_id="<ID>", goal_value=30000000)
reddit_ads_update_ad(ad_id="<ID>", configured_status="PAUSED")
```

Delete honors Reddit's archive-then-delete rule — `reddit_ads_delete_campaign` / `_delete_ad_group` / `_delete_ad` archive the entity if it isn't yet hard-deletable (ARCHIVED for 3+ hours), and permanently delete once eligible. Inspect the returned `configured_status` to see which happened, and tell the user when a hard delete will be possible.

## Campaign Workflow

Discover business/account → audit account → research (objective, targeting, budget, post) → resolve targeting IDs → confirm strategy → create campaign (`PAUSED`) → create ad group → find/create post → create ad → review with user → activate (`update_*` to `ACTIVE`).

## Known Limitations

| Issue | Workaround |
| --- | --- |
| No instant hard delete; "A campaign must be archived before it can be deleted" / "cannot be deleted until three hours after it was archived" | Expected — `delete_*` auto-archives (stops delivery); hard delete is possible ~3h after archiving. Re-run delete later to remove permanently. |
| Ad creation needs a `post_id` | Promote an existing community post (`list_posts`) or create one (`create_post`); only community posts can be promoted via the API. |
| Conversion ad groups rejected without a pixel | Set `conversion_pixel_id` (from `list_pixels`). Required for ad groups / CBO from 2026-07-13. |
| Campaign won't serve | Attach a `funding_instrument_id` (from `list_funding_instruments`); the account must be funded. |
| Lead-gen forms can't be created via API | Build the form in Reddit Ads Manager; the API only sets the `LEAD_GENERATION` objective. |
| Variant create fields (targeting, schedule, creative) aren't named params | Pass them through `input_data` (a dict), which is merged into the request body. |
| No flat ad-account list | Discover via `get_me → list_businesses → list_ad_accounts(business_id)`. |
| `"You cannot set invalid communities: {t5_...}"` | The `targeting.communities` array wants subreddit **names**, not the `t5_` IDs from `search_targeting`. Use the `name` field. |
| `"Cannot set start_time ... without ... campaign budget optimization"` | Campaign-level `start_time` is CBO-only. Omit it on non-CBO campaigns; set `start_time` on the ad group. |
| Ad group create: `"Input should be a valid datetime"` on `start_time` | `start_time` is **required** on ad groups — always pass it. |
| `bid_value` demanded despite `MAXIMIZE_VOLUME` | `bid_value` is always required for CPC/CPM/CPV/CPV6. Provide a cap or use `generate_bid_suggestion`. |
| `generate_bid_suggestion`: `"Additional fields not permitted"` / `"'start_time' is a required property"` | `duration` is an ISO start/end range (`{"start_time": ..., "end_time": ...}`), not `{"days": N}`. |
| Varnish **503** on create | Transient Reddit backend error. Verify no duplicate was created (`list_campaigns`) before retrying. |
| `effective_status` shows `PENDING_APPROVAL` right after create | Normal even for PAUSED entities (ad review) — it clears automatically; not a failure. |
| CONVERSIONS ad group: `"Conversion goal (optimization goal) is required"` / `"... is not valid for Conversions Campaigns"` | `optimization_goal` is required and, for non-CBO CONVERSIONS, must be `CLICKS` (SIGN_UP/PAGE_VISIT/PURCHASE are rejected for this campaign type). |
| `"'LINK' is not one of [CAROUSEL, IMAGE, TEXT, VIDEO]"` | Post `type` must be IMAGE/VIDEO/CAROUSEL/TEXT — there is no LINK type. |
| Ad create: `"Posts used for ads in Conversion campaigns cannot be free-form ad"` / `"Free form ads cannot have a click url"` | TEXT posts are free-form — use IMAGE/VIDEO/CAROUSEL for CONVERSIONS ads. |
| IMAGE post: `"Image is required"` | Put the asset in `content[].media_url` (an already-uploaded Reddit asset from `list_creative_assets`); a `post_url` alone is not enough. |
| Post: `call_to_action "Additional fields not permitted"` | `call_to_action` belongs **inside `content[]`**, not top-level, and uses display labels (`"Sign Up"`), not enum constants (`"SIGN_UP"`). |
| TEXT post: `"Post content must be empty for this post type"` | TEXT posts can't carry a `content[]` block — only IMAGE/VIDEO/CAROUSEL do. |
| Dollar amounts | Always micro-currency (×1,000,000). |

## Safety Rules

**Never:**

- Assume business, ad account, campaign, post, pixel, or targeting IDs — look them up or ask.
- Skip the account audit phase.
- Create campaigns, ad groups, or ads without explicit user approval.
- Set anything to `ACTIVE` without user consent.
- Pass dollar amounts instead of micro-currency.
- Create an ad without a valid `post_id`.
- Promise an instant permanent delete — Reddit requires ARCHIVED for 3+ hours first.
- Guess targeting IDs — resolve them with `reddit_ads_search_targeting`.
