# Performance Max campaigns (blueprint)

> **CRITICAL**: Always use the blueprint system for campaign creation. It validates locally, fills smart defaults, resolves locations, and rolls back on failure. Workflow: build blueprint → **preview** → explicit user approval → **create** ([discovery.md](../discovery.md) must be complete and the summary approved first).

## Performance Max Campaigns

**Preview**: `google_ads_preview_pmax_blueprint(blueprint={...})`
**Create**: `google_ads_create_from_pmax_blueprint(blueprint={...})`

```json
{
  "customer_id": "1234567890",
  "name": "PMax Campaign",
  "budget_amount_micros": 50000000,
  "bidding_strategy": "MAXIMIZE_CONVERSIONS",
  "status": "PAUSED",
  "location_names": ["United States"],
  "conversion_action_ids": ["1234567890"],
  "asset_groups": [
    {
      "name": "Asset Group 1",
      "headlines": ["Headline 1", "Headline 2", "Headline 3"],
      "long_headlines": ["Longer headline up to 90 characters"],
      "descriptions": ["Description 1", "Description 2"],
      "business_name": "Business Name",
      "final_urls": ["https://example.com"],
      "marketing_image_asset_ids": ["1234567890"],
      "square_marketing_image_asset_ids": ["1234567891"],
      "logo_asset_ids": ["1234567892"]
    }
  ]
}
```

> **CRITICAL (PMax)**: All three image asset types are REQUIRED with correct aspect ratios. The preview tool validates ratios before creation.

## Standalone PMax asset group (existing campaign)

Use `google_ads_asset_groups_create` (legacy name `google_ads_create_asset_group`) to add an asset group to an **existing** Performance Max campaign without running the full blueprint flow.

Required: `customer_id`, `campaign_id`, `final_urls`, text assets (`headlines`, `long_headlines`, `descriptions`, `business_name`), and all three image asset ID lists.

## Campaign dates (API v24)

Optional `start_date` / `end_date` on blueprints and campaign tools use `YYYY-MM-DD`. The backend encodes them to v24 wire fields `startDateTime` / `endDateTime` (`yyyy-MM-dd HH:mm:ss`). Omit both dates to let Google set schedule defaults.

Re-check [constraints.md](../constraints.md) (blueprint features, bidding strategies, technical rules) at each creation step.
