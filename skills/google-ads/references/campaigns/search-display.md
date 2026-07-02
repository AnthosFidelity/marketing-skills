# Search & Display campaigns (blueprint)

> **CRITICAL**: Always use the blueprint system for campaign creation. It validates locally, fills smart defaults, resolves locations, and rolls back on failure.

## Workflow: Preview → Confirm → Create

1. Build the blueprint JSON from research ([discovery.md](../discovery.md) must be complete and the summary approved).
2. Call the **preview** tool to validate and show the user what will be created.
3. Get explicit user approval.
4. Call the **create** tool.

## Search & Display Campaigns

**Preview**: `google_ads_preview_blueprint(blueprint={...})`
**Create**: `google_ads_create_from_blueprint(blueprint={...})`

```json
{
  "customer_id": "1234567890",
  "name": "Campaign Name",
  "budget_amount_micros": 50000000,
  "advertising_channel_type": "SEARCH",
  "bidding_strategy": "MAXIMIZE_CLICKS",
  "status": "PAUSED",
  "location_names": ["New York", "Los Angeles"],  // array in blueprints; note: google_ads_search_locations takes a single string for manual lookup
  "conversion_action_ids": ["1234567890"],
  "ad_schedules": [
    {"day_of_week": "MONDAY", "start_hour": 9, "end_hour": 17}
  ],
  "sitelinks": [
    {"link_text": "Free Trial", "final_urls": ["https://example.com/trial"]}
  ],
  "callouts": [{"callout_text": "Free Shipping"}],
  "ad_groups": [
    {
      "name": "Ad Group 1",
      "keywords": [
        {"text": "marketing software", "match_type": "PHRASE"},
        {"text": "competitor brand", "match_type": "EXACT", "negative": true}
      ],
      "ads": [
        {
          "headlines": ["Headline 1", "Headline 2", "Headline 3"],
          "descriptions": ["Description 1", "Description 2"],
          "final_urls": ["https://example.com"]
        }
      ],
      "audiences": [{"audience_id": "1234567890"}]
    }
  ]
}
```

For **Display** campaigns, set `"advertising_channel_type": "DISPLAY"` and use `display_ads` instead of `ads`:
```json
"display_ads": [{
  "headlines": ["Headline 1", "Headline 2", "Headline 3"],
  "long_headline": "Longer headline up to 90 characters",
  "descriptions": ["Description 1", "Description 2"],
  "business_name": "Business Name",
  "final_urls": ["https://example.com"],
  "marketing_image_asset_ids": ["1234567890"],
  "square_marketing_image_asset_ids": ["1234567891"],
  "logo_asset_ids": ["1234567892"]
}]
```

> **CRITICAL (Display)**: All three image asset types are REQUIRED:
> - `marketing_image_asset_ids` — landscape 1.91:1 (e.g. 1200×628).
> - `square_marketing_image_asset_ids` — square 1:1 (e.g. 1200×1200).
> - `logo_asset_ids` — square 1:1 or landscape 4:1.

Re-check [constraints.md](../constraints.md) (blueprint features, bidding strategies, technical rules) at each creation step.
