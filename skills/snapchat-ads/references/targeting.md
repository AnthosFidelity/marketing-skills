# Snapchat Ads: Targeting Lookups

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
