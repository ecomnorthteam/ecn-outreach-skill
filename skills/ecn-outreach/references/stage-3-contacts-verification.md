# Stage 3 — Contacts + Email Verification

## Pull decision-makers

`find_decision_makers` (Master Leads DB) on surviving domains, ≤500 domains/call:

```
{
  domains: [...],
  is_decision_maker: true,      // founders, CEOs, owners — the default and the ECN standard
  email_required: true,
  verify_emails: true,          // inline Emailable re-verify of stale (>60d) emails — we're sending soon
  limit: 2000
}
```

Read the response contract carefully:
- Report from `total_matching`, not `returned`.
- `would_yield_if_filters_relaxed` tells you how many more contacts dropping `is_decision_maker` or `email_required` would yield — quote these numbers when yield is short.
- `send_ready_filter.removed_post_fetch` shows verification attrition.

## Verification + bucketing

Emails returning from `find_decision_makers` with fresh verification are done. For any contact whose email is unverified or stale and wasn't covered inline, batch through `emailable_verify_emails` (≤500/call, ~$0.001/email — check budget headroom first for big batches).

Bucket every contact:

| Verification state | Bucket | Action |
|---|---|---|
| deliverable | **MAIN** | enroll in GetSales flow |
| risky (catch-all, role-based, low score) | **RISKY** | keep as separate list — never enrolled this run; user decides later |
| undeliverable | dropped | counted in report only |
| no email found | dropped (email channel) | if run includes LinkedIn, may still enroll for LinkedIn-only path — ask user |

Persist both lists to the run scratch dir (`./runs/<date>-<campaign>/main-contacts.json`, `risky-contacts.json`) with: first_name, last_name, title, email, verification state, company name, domain, linkedin URL if present, company platform + est. revenue (from Stage 2 — useful for personalization variables later).

## Yield shortfall loop

If MAIN bucket < target lead count:
1. Quote `would_yield_if_filters_relaxed` numbers.
2. Offer, in order: (a) include non-decision-maker contacts at surviving companies, (b) include risky bucket (warn: bounce risk), (c) widen Stage 1 pull and re-run the pipeline on the delta. AskUserQuestion — never auto-pick.

## One contact per company

Default: enroll only the single best contact per company (founder > CEO > owner > co-founder ranking; tie-break by verified email quality). Multiple contacts from one company in the same cold sequence looks spammy and burns the account. If the user explicitly wants multi-threading, allow max 2 and stagger is their problem to manage in GetSales.

## Checkpoint output format

```
## Contact funnel
Companies in:        N
Contacts matched:    N (total_matching)
Decision-makers:     N
With email:          N
Deliverable (MAIN):  N
Risky (held):        N
Undeliverable:       N
No email:            N
```
Plus 10-row sample table: name | title | company | email | status. Then AskUserQuestion: proceed to campaign build / relax filters / widen pull.
