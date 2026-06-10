# Stage 1 — Lead Pull (StoreLeads-first, Apollo for people)

All tools live on the `Master Leads DB` MCP server (`mcp__claude_ai_Master_Leads_DB__*`).

**Field-tested 2026-06-10:** Apollo company search for "ecommerce brands" is noise-dominated (keyword_tags matched banks and Government of Canada; `technologies:['Shopify']` matched recruiters; 50 credits → 3 usable brands). StoreLeads search of the same target produced ~100% real stores at $0. Hence:

## Route by business unit

- **events BU (DTC brands)** → **PATH A: StoreLeads-first** (default, free)
- **sponsor BU (B2B vendors)** → **PATH B: Apollo-first** (vendors aren't stores; StoreLeads can't find them)

## PATH A — StoreLeads-first (events BU)

`storeleads_search_domains` — free, every result is a real online store, auto-upserts.

```
{
  platform: 'shopify',          // or omit for all platforms
  country_code: 'CA',           // ISO-2, one per call — multi-country = multiple calls
  page_size: 50, page: N,
  sort: '-estimated_sales_yearly'   // NOTE: backend appears to order by revenue regardless; verify per page
}
```

**Verified gotchas (do not rediscover these):**
1. `fields` projection: domain comes back in **`name`** (not `domain`), store title in `title`. If you project fields, ALWAYS include `name` and `estimated_sales_yearly`.
2. `fields` projection **breaks the master-DB upsert** — all rows skip as `invalid_brand_domain`. For pulls you want persisted: do one call WITHOUT `fields` per page you keep (big response — immediately jq the saved overflow file rather than reading it), or accept no-upsert on exploratory pages and re-fetch keepers without projection.
3. The `min/max_estimated_monthly_revenue_usd` toggleables divide `estimated_sales_yearly` by 100 (treat-as-cents bug) → real thresholds fail. **Filter revenue client-side**: `estimated_sales_yearly` is annual USD; 7-figure = 1M–10M, 8-figure = 10M+.
4. No city filter — filter client-side on the returned `city` field (GTA = Toronto, Mississauga, Markham, Vaughan, Milton, Oakville…).
5. Pages are roughly revenue-descending. For 7-figure targets, top pages = 8-9 figure giants; probe pages ~8-15 and adjust by observed values.

Loop pages until enough brands pass {revenue band, city, category} client-side filters — target 1.5-2× the lead count (StoreLeads results barely shrink in Stage 2, unlike Apollo's 90%+).

Categories use Google taxonomy strings (`/Beauty & Fitness/...`, `/Apparel`, `/Food & Drink/...`) — usable both as `categories` param and client-side.

## PATH B — Apollo-first (sponsor BU, or events fallback)

`apollo_search_companies` ($0.05/result — charged for noise too):

| Intake | Argument | Notes |
|---|---|---|
| Vendor category | `industry_keywords: ['logistics','payments',…]` + `keyword_tags: ['ecommerce brands','dtc','merchants']` | keyword_tags alone is dangerously loose — never the only filter |
| Geo | `countries: ['Canada']`, `locations: ['Toronto, Ontario']` | |
| Size | `employee_ranges: ['11,50','51,200']` | constrains Apollo intent — use it; min/max revenue args DON'T constrain Apollo, only post-filter DB rows |
| Dedup | `exclude_already_contacted_days` + `business_unit` | |
| Count | `limit` ≈ 3× target | cap 500 |

Report `apollo_cost.credits_used` to the user after every call.

## Decision-makers (both paths)

1. `find_decision_makers({domains, email_required:true, verify_emails:true})` — FREE, master DB. Read `total_matching` not `returned`; `would_yield_if_filters_relaxed` for shortfall options.
2. For domains with no DB contacts: `apollo_find_decision_makers({domains, person_titles:['CEO','Founder','Co-Founder','Owner','President'], max_results_per_domain:2, reveal_emails:true})` — charges credits only on reveals; Apollo often has nothing for small DTC brands (returns 0-credit empty).
3. Still missing + channels include LinkedIn → lead can proceed LinkedIn-only (flag it). Optionally offer `fullenrich_*` waterfall as paid last resort — ask first.

## Output

Dedup by domain across calls. Persist to `./runs/<date>-<campaign>/stage1-companies.json`: `{name/domain, title, city, country, platform, estimated_sales_yearly, categories}`. Check `get_daily_budget_status` before any paid step.
