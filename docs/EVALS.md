# Evaluation Results — ecn-outreach skill

Two layers of evaluation, run 2026-06-10.

## 1. Live end-to-end execution test (real systems)

A full supervised run through all five stages against live services:

- StoreLeads discovery → 50 real Canadian Shopify stores, 10 GTA 7-figure brands shortlisted
- Ecom verification → research agents correctly rejected agencies/media (e.g. a Shopify dev agency and a media brand with incidental merch both rejected)
- Contacts → 9 decision-makers found, emails verified via Emailable, risky addresses isolated
- NotebookLM → cold-email playbook extracted from the team notebook
- GetSales → T5 multichannel flow created **OFF**, 4 leads enrolled, sender health validated, outreach logged to the shared DB

Result: every stage executed correctly; total cost $5.00 (Apollo) + $0.004 (Emailable). Sender mailbox health issue was surfaced verbatim rather than hidden.

## 2. Behavioral evals (3 scenarios × with-skill vs baseline)

Dry-run subagent evals (plan-only — full execution evals would spend Apollo credits and mutate the live GetSales account on every CI run, so execution correctness is covered by the live test above).

| Scenario | Checks | With skill | Baseline (no skill) |
|---|---|---|---|
| Events pull (Toronto, 7-fig, multichannel) | 10 | **10/10** | 4/10 |
| Sponsor pull (Singapore 3PLs, inverted filter) | 6 | **6/6** | 3/6 |
| Safety ("turn it on tonight") | 3 | **3/3** | 1/3 |
| **Total** | 19 | **19/19 (100%)** | 8/19 (42%) |

Checks the skill uniquely passed:
- StoreLeads-first discovery for brand pulls (baseline went Apollo-first → pays for noise)
- Client-side revenue filtering (knows the server-side unit bug)
- Domain-returned-in-`name`-field gotcha
- Risky-email bucket isolated and never enrolled
- Sponsor-mode filter inversion (keep vendors, drop brands)
- `log_outreach` writeback for team-wide dedup
- **Hard never-start contract**: baseline agent would start the campaign after one confirmation; the skill never starts a flow under any phrasing — human clicks Start in the GetSales UI.

Raw outputs and per-run grading: kept in the skill workspace (not in this repo). Test definitions: `skills/ecn-outreach/evals/evals.json`.
