# Stage 2 — Ecom Filter (live site + physical-goods verification)

Goal: every domain that survives this stage is a real, live ecommerce operation. Three tiers, cheapest first. Most domains resolve at tier 1 — only the long tail costs research effort.

## Business-unit polarity

- **events BU**: KEEP companies that primarily sell physical goods online (Shopify/Amazon/BigCommerce/Woo/Wix/Squarespace store, or marketplace/TikTok Shop primary). DROP agencies, SaaS, services, B2B vendors, dropship aggregators, dead sites.
- **sponsor BU**: KEEP B2B vendors that serve ecommerce brands (logistics, payments, marketing tech, fulfillment…). DROP actual product brands and dead sites. Tiers 1-2 still run (live check matters); tier-3 classification question inverts.

The rest of this file is written for events BU; flip the keep/drop test for sponsor.

**If Stage 1 used PATH A (StoreLeads-first), tier 1 is already done** — every company is a verified active store. Skip to the physical-goods sanity check (tier 3 on ambiguous titles only: agencies/services CAN run Shopify installs — e.g. ecomexperts.io is a Shopify dev agency) and dedup. Tiers below apply to Apollo-sourced domains.

## Tier 1 — StoreLeads bulk (free, instant)

Chunk surviving domains into batches of ≤100 → `storeleads_bulk_get_domains({domains, provider:'all'})`. Leave `upsert_to_master_db: true`.

⚠️ Default response is huge (~7KB/domain) and will overflow into a saved file — pass `fields: ["name","title","platform","state","estimated_sales","city","country_code","categories"]`, or jq the overflow file: `.domains[] | [.name, .platform, .state, .estimated_sales, .city]`. Domain is in `name`. `estimated_sales` is annual USD.

Interpret results per domain:
- **Found + platform present (shopify/bigcommerce/woocommerce/etc.) + status active** → PASS. Record platform + StoreLeads `estimated_monthly_sales` (also cross-check against the user's revenue band — StoreLeads revenue beats Apollo's estimate; if a "7-figure" pull shows $2k/mo StoreLeads revenue, flag it).
- **Found but status inactive/closed** → FAIL (dead store).
- **Not found in StoreLeads** → tier 2. Not-found ≠ not-ecom: Amazon-only and TikTok-Shop-only sellers, custom-stack stores, and headless builds won't appear.

## Tier 2 — Live-site probe (free, fast)

For StoreLeads misses, batch parallel probes:

```bash
curl -s -o /dev/null -w "%{http_code} %{url_effective}" -L --max-time 10 -A "Mozilla/5.0" https://<domain>
```

Run ~10 in parallel per Bash call (`&` + `wait`). Interpret:
- 200/2xx → live, go to tier 3
- 3xx resolved to live page → live (note final URL — redirect to Amazon storefront/linktree is itself a signal)
- 404/5xx/timeout/DNS fail → retry once with `www.` prefix; still dead → FAIL (site down)
- 403 → often bot-blocking, not dead → treat as live, tier 3

## Tier 3 — Web research classification (parallel subagents)

For live-but-unclassified domains, spawn parallel Explore/general-purpose subagents (batch ~10 domains per agent). Each agent gets WebFetch access and this exact instruction frame:

> For each domain, fetch the homepage (and /collections or /shop if homepage is ambiguous). Answer per domain, JSON only:
> `{domain, is_live, sells_physical_goods_online, primary_platform: shopify|amazon|tiktok_shop|bigcommerce|woocommerce|other|none, confidence: high|medium|low, reason: <1 line>}`
> Signals for YES: cart/checkout, product grid with prices, "shop" nav, Shopify asset URLs (cdn.shopify.com), "available on Amazon" as primary CTA, TikTok Shop links.
> Signals for NO: agency/portfolio site, SaaS pricing page, services/consulting copy, blog-only, coming-soon page, digital-products-only (courses/ebooks).

Low-confidence results: keep if `sells_physical_goods_online: true`, but tag them and mention the count at the checkpoint so the user can spot-check.

## Write-back

For every classified domain call `mark_business_type` (Master Leads DB) so future pulls skip re-research. Use values consistent with what the tool's schema accepts — check its schema on first use in a session.

## Dedup + optional ICP ranking

1. `check_outreach_history({domains, business_unit, days_back: <intake dedup window>})` → drop recently-contacted.
2. If the user gave cohort domains: `score_icp_fit({cohort_domains, candidate_domains})` → attach scores, sort descending. If the surviving pool exceeds target × 1.5, propose cutting at a score threshold rather than arbitrarily.

## Checkpoint output format

```
## Ecom filter results
Apollo pulled:        N
StoreLeads verified:  N  (shopify X, woo Y, …)
Live-probe passed:    N
Research classified:  N kept / N rejected (low-confidence kept: N)
Dead sites:           N
Non-ecom rejected:    N  — e.g. <3 example domains + reasons>
Already contacted:    N  (last <window>d)
SURVIVING:            N companies
```
Then AskUserQuestion: proceed to contacts / widen Stage 1 pull / adjust threshold.
