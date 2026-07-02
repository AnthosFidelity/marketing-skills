---
name: google-ads
description: Plan and create new Google Ads campaigns end-to-end via the Hyper MCP. Use when the user wants to launch Search, Display, or Performance Max campaigns, or mentions Google Ads, AdWords, search ads, display ads, Performance Max, PMax, PPC, Google campaigns, Google blueprint, search term reports, negative keywords, or manager accounts (MCC). For ongoing optimization, search-term cleanup, conversion diagnosis, or restructuring existing accounts, defer to a future `google-ads-operator` sibling skill.
---

# Google Ads

Strategic guide for building new Google Ads campaigns. Research first, consult intelligently, validate everything, and use the blueprint flow for controlled creation.

## Requirements

- **Hyper MCP installed and connected.** [https://app.hyperfx.ai/mcp](https://app.hyperfx.ai/mcp)
- **Google Ads integration connected** at [https://app.hyperfx.ai/integrations](https://app.hyperfx.ai/integrations).

If `google_ads_list_accounts` is not in the tool list, stop and tell the user to enable Hyper MCP and connect Google Ads.

## Out of scope

- **Ongoing optimization** (search-term cleanup, bid adjustments, ad testing, restructuring existing campaigns) → defer to a future `google-ads-operator` skill.
- **Creative generation** (headlines, descriptions, images) → [`ad-creative-generation`](../ad-creative-generation).
- **Cross-platform campaign launches** → use this skill for Google, then invoke `meta-ads` / `tiktok-ads` separately.

## Tool surface

| Tool | Purpose |
| --- | --- |
| `google_ads_list_accounts` | Discover accessible accounts (and MCC sub-accounts). |
| `google_ads_execute_gaql` | Run a GAQL query (conversion actions, search term reports, etc.). |
| `google_ads_search_locations` | Resolve human-readable location names to IDs. |
| `google_ads_list_assets`, `google_ads_upload_image_asset` | Manage image assets for Display / PMax. |
| `google_ads_preview_blueprint`, `google_ads_create_from_blueprint` | Validate + create Search/Display campaigns. |
| `google_ads_preview_pmax_blueprint`, `google_ads_create_from_pmax_blueprint` | Validate + create Performance Max campaigns. |

## Rules that must never be forgotten

> **BUDGETS IN MICROS**: $50/day = 50,000,000 micros. Never pass dollar amounts directly.

> **ALWAYS START PAUSED**: Create campaigns with `status="PAUSED"`. Activate only after explicit user approval.

> **PREVIEW BEFORE CREATE**: Always use the blueprint system — build the blueprint JSON, call the **preview** tool, get explicit user approval, then call the **create** tool. Never skip the preview step. The blueprint validates locally, fills smart defaults, resolves locations, and rolls back on failure.

> **NEVER INVENT**: Don't assume URLs, budgets, or location IDs. Don't skip research. Don't invent keywords or copy — creative comes only from the actual site. Don't build without user buy-in.

> **GAQL GUARDRAILS**: Never join `ad_group_criterion` with `search_term_view`. Never query more than 5 sub-accounts in a single batch.

See [references/constraints.md](references/constraints.md) for the full constraint set.

> **All reference files live in `references/`.** Read them at `references/<file>` (e.g. `references/discovery.md`). They are not in the same directory as this SKILL.md.

## Core process

Every campaign build follows this sequence. Do not skip steps.

1. **Identify the goal** — new campaign, reporting/analysis, or both?
2. **Check the routing table** and read the referenced files before calling any tools
3. **Discovery is mandatory for creation** ([references/discovery.md](references/discovery.md)) — account setup, site scan, conversion tracking check, market research, consultation
4. **Present the pre-creation summary** and wait for explicit approval
5. **Build blueprint → preview → user approval → create (PAUSED)**, re-checking [references/constraints.md](references/constraints.md) at each step
6. **Activate only when the user approves**

Full workflow: Initial Setup → Research (site + GAQL + market) → Analyze (goals, audience, keywords) → Consult (options + trade-offs) → Recommend (structure + bids) → Confirm (summary approved) → Build Blueprint → Preview → User Approval → Create (PAUSED) → Activate (post-approval).

## Routing table

| The user wants to… | Read these files first |
|---|---|
| Create a Search or Display campaign | [references/discovery.md](references/discovery.md) → [references/campaigns/search-display.md](references/campaigns/search-display.md) |
| Create a Performance Max campaign | [references/discovery.md](references/discovery.md) → [references/campaigns/pmax.md](references/campaigns/pmax.md) |
| Add an asset group to an existing PMax campaign | [references/campaigns/pmax.md](references/campaigns/pmax.md) |
| Run a search term report / GAQL query / analyze performance | [references/reporting.md](references/reporting.md) |
| Work across an MCC / many sub-accounts | [references/mcc.md](references/mcc.md) → [references/reporting.md](references/reporting.md) |
| Goal not yet clear | [references/discovery.md](references/discovery.md) — discovery clarifies the goal |
