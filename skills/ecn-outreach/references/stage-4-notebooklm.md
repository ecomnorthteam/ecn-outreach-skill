# Stage 4 — NotebookLM Cold-Email Best Practices

The expertise source is a NotebookLM notebook curated specifically for cold email marketing:

**Notebook URL:** `https://notebooklm.google.com/notebook/e8fc5293-16dd-41d3-8bc9-d6e7107aa68d`

## How to query it

Use the **notebooklm** skill (`~/.claude/skills/notebooklm/`). Key mechanics from that skill — always go through the wrapper:

```bash
cd ~/.claude/skills/notebooklm
python3 scripts/run.py auth_manager.py status          # check auth first — NOTE: python3, plain `python` doesn't exist on this Mac
python3 scripts/run.py ask_question.py \
  --question "<question>" \
  --notebook-url "https://notebooklm.google.com/notebook/e8fc5293-16dd-41d3-8bc9-d6e7107aa68d"
```

If not authenticated: `python3 scripts/run.py auth_manager.py setup` opens a visible browser for Google login — tell the user a browser window will open and they need to log in once. A stale-but-authenticated state ("state is N days old") usually still works — try a query before forcing re-auth.

If this notebook isn't yet in the library (`notebook_manager.py list`), add it after first successful query (name: "Cold Email Marketing Expertise", topics: cold email, outreach, deliverability, sequences).

## What to ask

Each browser query is slow (~1-2 min) — ask few, dense questions. **Field-tested: one comprehensive question covering sequence + copy rules + cadence usually extracts everything; a second question often returns near-identical content** (notebook sources are general cold-email). Default to 1 dense question; add a second only for a genuinely different topic (e.g. deliverability setup). Templates:

1. **Sequence architecture:** "For a cold email campaign to [ecommerce founders of 7-figure DTC brands / B2B vendors serving ecommerce] with the goal of [goal], what sequence structure do you recommend — number of emails, days between each, and the job of each email? Include guidance on the breakup email."
2. **Copy principles:** "What are the most important rules for subject lines, opening lines, body length, personalization, and CTA in cold emails to busy founders? Give specific dos and don'ts with examples where the sources include them."
3. **Deliverability/setup:** "What sending practices protect deliverability for cold email — daily volume per sender, ramp-up, plain-text vs HTML, links in first email, spintax/variation?" (Ask only on the first run or when senders changed; cache the answer in the run dir for reuse.)
4. **Goal-specific** (optional): event invite framing, sponsor pitch framing, etc., depending on the campaign goal.

## Synthesize → campaign playbook

Condense answers into `./runs/<date>-<campaign>/playbook.md`:

```
# Campaign Playbook — [campaign name]
## Sequence skeleton
Step 1 (day 0): job, angle
Step 2 (day +N): …
…
## Copy rules
- subject line rules
- opening / personalization strategy (which lead fields to use: first_name, company, platform, niche…)
- body length, tone, CTA per step
## Sending rules
- volume/ramp guidance relevant to GetSales schedule settings
```

This playbook is the single source Stage 5 writes copy from. Where notebook guidance conflicts with Ecom North context (e.g. generic SaaS examples), adapt the principle, keep the structure.

## Fallback

NotebookLM is browser automation — it can break (auth expiry, UI changes, headless env). On failure:
1. Report the exact error.
2. AskUserQuestion: fix auth now (user logs in) / proceed on Claude's own cold-email judgment / abort.
3. If proceeding without the notebook, mark the playbook header `SOURCE: Claude judgment — NotebookLM unavailable` so the user knows at review time.
