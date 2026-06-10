---
name: ecn-outreach
description: End-to-end Ecom North outreach pipeline — pull ecommerce leads from Apollo (via Master Leads DB MCP), verify each company is a live ecommerce brand selling physical goods (StoreLeads + web research), verify emails, pull cold-email best practices from the NotebookLM cold-email notebook, then build a ready-to-launch campaign (flow + contacts + senders) in GetSales that the human just reviews and turns on. Use this skill whenever the user wants to pull leads, run a lead pull, build/spin up/launch an outreach or cold-email campaign, find ecommerce brands to contact, prospect for the summit/events/sponsors, filter non-ecommerce leads, or do anything that chains Apollo → research → GetSales — even if they only mention one stage by name.
---

# ECN Outreach Pipeline

Five-stage pipeline that turns ICP parameters into a paused, fully-loaded GetSales campaign. The human's only job at the end is to review the flow in GetSales and click start.

```
Stage 0: Intake          → collect run parameters (checkpoint)
Stage 1: Apollo pull     → companies via Master Leads DB MCP
Stage 2: Ecom filter     → StoreLeads + live-site check + web research (checkpoint)
Stage 3: Contacts        → decision-makers + Emailable verification (checkpoint)
Stage 4: NotebookLM      → cold-email best practices for this campaign type
Stage 5: GetSales build  → flow + copy + enroll leads + log outreach (checkpoint)
```

**Who we are (bake into all copy and ICP judgment):** Ecom North is a global eCommerce collective — flagship summits (Singapore, LA, Toronto, Melbourne), founder dinners, retreats, The Warehouse Podcast, and a 2,000+ brand community. Two audiences: (1) DTC/ecommerce **founders** scaling 7-8 figure brands → events business unit; (2) B2B **service providers** (payments, logistics, marketing platforms) → sponsor business unit.

## Core principles

1. **Checkpoint, don't assume.** At every stage marked (checkpoint), summarize what happened with real counts, then use AskUserQuestion before proceeding. The user may change parameters mid-run — accommodate, re-run that stage, continue.
2. **Never start a flow.** GetSales flows are created in `off` status. Never call `control_flow({action:'start'})`. The human starts it in the UI. This is the safety contract of the whole pipeline.
3. **Overshoot then filter.** Apollo results always contain non-ecommerce noise. Pull 2.5-3× the target lead count of companies so that after ecom filtering + dedup + email verification you still hit the target.
4. **Write back everything.** Classifications go back to Master DB (`mark_business_type`), enrollments go to `log_outreach`. The DB is the memory that makes the next pull cheaper and dedup correct.
5. **Free before paid.** StoreLeads and Apollo company search are free; Emailable charges per email; some enrichments charge. Check `get_daily_budget_status` before the run and before any paid bulk step.
6. **If a tool fails or data looks wrong, say so.** Don't fabricate counts. Surface the error and ask the user how to proceed.

## Stage 0 — Intake

Read `references/stage-1-apollo-pull.md` for parameter→tool mapping details.

Collect via AskUserQuestion (batch into 1-2 question rounds, max 4 questions each). If the user already specified a parameter in their request, don't re-ask it — confirm it in the summary instead.

**Required parameters:**

| Parameter | Options / format | Default |
|---|---|---|
| Business unit | `events` (founders/brands) or `sponsor` (B2B vendors) | events |
| Target lead count | number of send-ready contacts wanted | 100 |
| Revenue band | 7-figure ($1M-$10M/yr ≈ $83k-$833k/mo), 8-figure ($10M+/yr ≈ $833k+/mo), smaller, or custom | 7-figure+ |
| Industry/niche | free keywords: beauty, CPG, supplements, apparel, pets, home… | any ecom |
| Geography | countries and/or cities — usual markets: Canada, USA, Singapore, Australia (summit cities: Toronto, LA, Singapore, Melbourne) | ask |
| Channels | email only / email + LinkedIn | ask |
| Dedup window | exclude companies contacted in last N days | 90 |

**Optional parameters (offer, don't force):**
- Employee range buckets (`1,10` / `11,50` / `51,200`…)
- Tech stack filter (e.g. Klaviyo, ReCharge — ALL must match)
- Founded year range
- ICP cohort domains (past attendees/customers) → enables `score_icp_fit` ranking in Stage 2
- Campaign goal free-text (e.g. "invite to Toronto Summit Oct 2026", "book sponsor discovery calls") — feeds Stage 4 + 5 copy

Then call `get_daily_budget_status` (Master Leads DB). If spend headroom looks too small for the run (rough cost: Emailable ≈ $0.001/email × ~1.5× target count; enrichment varies), warn the user before continuing.

## Stage 1 — Lead pull

Read `references/stage-1-apollo-pull.md` before executing — it has the field-tested gotchas (unit bugs, field projections) that cost real money to rediscover.

Summary: **events BU → StoreLeads-first** (`storeleads_search_domains`, free, ~100% real stores; revenue/city filtered client-side); **sponsor BU → Apollo-first** (`apollo_search_companies`, $0.05/result, intent-level filters only). Both auto-upsert to master DB. Then decision-makers: master DB free lookup → Apollo paid fill for gaps.

Report: companies found, cost so far, sample of 10 (name, domain, est. revenue), sparse-result relax suggestions.

## Stage 2 — Ecom filter (checkpoint)

Read `references/stage-2-ecom-filter.md` before executing.

Goal: keep only companies that (a) have a live website and (b) primarily sell physical goods online (Shopify, Amazon, BigCommerce/Woo/etc., or TikTok Shop). For `sponsor` business unit the test inverts: keep B2B vendors serving ecommerce, drop actual brands — the reference file covers this.

Three-tier funnel (cheap → expensive). StoreLeads-first pulls skip tier 1-2 (already verified stores) — only tier-3 spot-check ambiguous titles (agencies/media CAN run Shopify installs):
1. **StoreLeads bulk** (`storeleads_bulk_get_domains`, 100 domains/call, free) — resolves platform, status, est. revenue for most real stores instantly.
2. **Live-site check** for StoreLeads misses — parallel `curl` HEAD/GET probes.
3. **Web research agents** for still-ambiguous domains — parallel subagents classify via homepage fetch.

Write classifications back with `mark_business_type`. Then `check_outreach_history` dedup (belt-and-suspenders even though Stage 1 excluded). If ICP cohort provided, rank with `score_icp_fit`.

Checkpoint: report funnel table (pulled → live → ecom-verified → deduped), rejected examples with reasons, and ask: proceed / widen pull / adjust filters.

## Stage 3 — Contacts + email verification (checkpoint)

Read `references/stage-3-contacts-verification.md` before executing.

Summary: `find_decision_makers` on surviving domains (founder/owner/CEO default), `verify_emails: true` for stale emails. Then `emailable_verify_emails` for anything still unverified. Bucket results:
- **deliverable** → main enrollment list
- **risky** → separate list, NOT enrolled — presented to user for manual decision
- **undeliverable / no email** → dropped (kept in report)

If contact yield < target, loop: relax `is_decision_maker`, or return to Stage 1 with widened params (ask first).

Checkpoint: report contact funnel, show 10 sample contacts (name, title, company, email status), confirm the final enrollment list size.

## Stage 4 — NotebookLM best practices

Read `references/stage-4-notebooklm.md` before executing.

The cold-email expertise notebook: `https://notebooklm.google.com/notebook/e8fc5293-16dd-41d3-8bc9-d6e7107aa68d`

Invoke the **notebooklm** skill (it lives at `~/.claude/skills/notebooklm/` — use its `run.py` wrapper, check auth status first). Ask the notebook 2-4 targeted questions shaped by this run's campaign goal, audience, and channels — e.g. sequence structure + cadence, subject lines for this audience, personalization strategy, follow-up/breakup approach. Synthesize answers into a short **campaign playbook** (sequence skeleton, copy guidelines, cadence) that drives Stage 5.

If NotebookLM auth fails or the browser automation breaks, tell the user and offer: fix auth now, or proceed with Claude's own cold-email judgment (flag that notebook guidance was skipped).

## Stage 5 — GetSales build (checkpoint)

Read `references/stage-5-getsales.md` before executing. This is the most mechanical stage — follow the reference exactly.

Summary of the build order:
1. `manage_sender_profiles({action:'list'})` → present senders to user → user picks (AskUserQuestion).
2. Pick structure: email-only → custom email nodes; email+LinkedIn → `list_flow_templates` → `get_flow_template` for closest T1-T7 match.
3. Write copy per the Stage 4 playbook. Discover variables via `crm.list_template_variables` first — use only variables that exist.
4. `create_flow` with nodes + sender_profiles + schedule. Flow lands in `off`. Relay the returned `post_build_summary` + `graph_ascii` verbatim. Surface any `health_issues[]` on senders.
5. Enroll: `enroll_leads_in_flow` for <100 leads; `bulk_import_to_flow` (CSV path) for larger. Flow is off, so enrolling is safe — but still confirm with the user first, noting leads activate when the human starts the flow.
6. `log_outreach` to Master DB (`external_system:'getsales'`, correct `business_unit` + `channel`, status `queued`).

Final checkpoint: hand the user the flow URL, sender health summary, enrolled count, risky-list location, and the one remaining human action: **review in GetSales and click start.**

## Run report

End every run with this exact structure:

```
# ECN Outreach Run — [date] — [campaign name]
## Parameters
## Funnel
Apollo pulled → live sites → ecom-verified → deduped → contacts found → deliverable → enrolled
(numbers at each step)
## Campaign
Flow name, URL, senders, sequence summary, schedule
## Awaiting human
- Review flow in GetSales → click start
- Risky-email list: [N contacts, where to find]
## Costs
Credits/spend consumed this run (from get_daily_budget_status delta)
```

## Failure & edge handling

- **Sparse Apollo results** → report `would_yield_if_filters_relaxed`-style info, propose which filter to drop, ask.
- **MCP down** (Master Leads DB / GetSales unreachable) → stop, report, never substitute fabricated data. Apollo direct MCP (claude.ai connector) is the fallback for Stage 1 only — ask before switching.
- **Budget cap hit mid-run** → pause at next stage boundary, report spend, ask.
- **User changes mind mid-run** → parameters are not sacred; re-run the affected stage and continue downstream. State clearly which stages are being re-run.
- **Resume**: if a previous run was interrupted, data is in Master DB (companies upserted, classifications marked). Re-running Stage 1 with identical params is cheap; Stages 2-3 skip already-classified/verified records automatically via the DB write-backs.
