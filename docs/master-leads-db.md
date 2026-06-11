# Master Leads DB — Connector Guide

The Master Leads DB is Ecom North's hosted lead-intelligence server. It is a **remote MCP connector** — it runs in the cloud, there is nothing to download or install on your machine. You connect your claude.ai account to it once, and the tools appear in every Claude Code session automatically.

> Note: the server's source code is not in this repo — it's maintained centrally. This doc covers getting access and what's inside.

## Getting access

1. Ask your team admin to share the Master Leads DB connector with your claude.ai account / the Ecom North workspace.
2. In **claude.ai → Settings → Connectors**, the connector appears as **"Master Leads DB"**. Add/enable it.
3. First use triggers a sign-in (OAuth). Use your Ecom North account.
4. Verify: in Claude Code, ask "check the daily lead budget" — you should get a spend-vs-cap answer, not an error.

## What's inside (what your pulls are actually using)

| Capability | Tools (prefix `Master Leads DB`) | Cost |
|---|---|---|
| Store discovery — 4M+ real online stores searchable by platform/country/revenue/category | `storeleads_search_domains`, `storeleads_bulk_get_domains`, … | **Free** (team subscription) |
| Company discovery — Apollo's 30M+ company DB | `apollo_search_companies` | $0.05/result |
| Find founders/CEOs at companies | `find_decision_makers` (free, our DB) → `apollo_find_decision_makers` (paid fill) | free / credits |
| Email verification | `emailable_verify_emails` | ~$0.001/email |
| Shared outreach history — team-wide "who did we already contact" | `check_outreach_history`, `log_outreach` | free |
| ICP scoring vs winner cohorts | `score_icp_fit` | free |
| Budget guardrails | `get_daily_budget_status`, `get_credit_burn` | free |

Everything found is automatically saved into our shared master database, so each pull makes the next one faster and cheaper.

## House rules

- **Always log outreach.** The ecn-outreach skill does this automatically (`log_outreach` after enrolling leads in GetSales). If you push leads to any sending tool manually, log it — otherwise teammates will re-contact the same brands.
- **Don't bulk-reveal Apollo emails casually** — each reveal costs credits. The skill asks before paid steps; keep that habit in manual use.
- Budget caps are enforced server-side per day. If a run stops with a budget message, that's the guardrail working — talk to your team admin to raise it.

## Field-tested API quirks (for anyone using the tools manually)

These are already baked into the ecn-outreach skill — listed here for manual/advanced use:

1. `storeleads_search_domains`: store domain is returned in the **`name`** field; `estimated_sales_yearly` is **annual USD**; the built-in revenue min/max filters mis-handle units — filter revenue yourself from raw values; there is no city filter — filter on the returned `city`.
2. `storeleads_search_domains` with a `fields` projection **does not save results to the master DB** (rows skip as `invalid_brand_domain`). Re-fetch keepers without `fields` if you want them persisted.
3. `storeleads_bulk_get_domains` default response is enormous — always pass a `fields` list.
4. `apollo_search_companies`: the revenue min/max arguments do **not** narrow the Apollo search itself (you pay for noise) — constrain with `employee_ranges` and `technologies` instead; `keyword_tags` alone is dangerously loose.
5. `find_decision_makers`: report counts from `total_matching`, not `returned`; `would_yield_if_filters_relaxed` tells you what loosening filters would add.
