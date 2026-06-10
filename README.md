# ECN Outreach — End-to-End Lead Pull & Campaign Builder

**Internal tool for the Ecom North team.** This is a Claude Code "skill" that runs our entire cold-outreach pipeline in one conversation: it finds real ecommerce brands (or sponsors), verifies them, finds the founders, checks their emails, pulls our cold-email best practices from NotebookLM, and builds a ready-to-launch campaign in GetSales.

**Your only job at the end: open the campaign in GetSales, review it, and click Start.** The tool never sends anything by itself — every campaign it builds is created in the OFF state.

---

## What it actually does (plain English)

```
You say: "Pull 100 beauty brand founders in Toronto and build a Summit invite campaign"

Stage 0  →  Claude asks you a few questions (how many leads, which market,
            email only or email + LinkedIn, which campaign goal)
Stage 1  →  Finds real online stores (StoreLeads database — free) or B2B
            vendors (Apollo) matching your filters
Stage 2  →  Verifies each company: website live? actually selling physical
            goods online? Filters out agencies, recruiters, dead sites
Stage 3  →  Finds the founder/CEO at each company and verifies their email.
            Sketchy emails go to a separate "risky" list — never contacted
Stage 4  →  Reads our cold-email expertise notebook (NotebookLM) and writes
            a campaign playbook: sequence, timing, copy rules
Stage 5  →  Builds the campaign in GetSales: sequence + copy + your chosen
            sender + all the leads enrolled. Campaign stays OFF.

You      →  Review in GetSales → click Start. Done.
```

At every stage Claude pauses, shows you the numbers, and asks before moving on. You can change your mind mid-run ("actually make it 8-figure brands only") and it adapts.

---

## Who can use this

Anyone on the Ecom North team with the prerequisites below set up. No coding needed — you talk to it in plain English inside Claude Code.

---

## Prerequisites (one-time setup)

You need all four of these before the skill will work end-to-end:

### 1. Claude Code
The Anthropic CLI/desktop app. Install: https://claude.com/claude-code
Sign in with your Ecom North Claude account.

### 2. Master Leads DB connector (claude.ai)
Our hosted lead-database server. It wraps Apollo (paid lead data), StoreLeads (free store database), Emailable (email verification), and our shared outreach history (so two people never cold-email the same brand twice).

- This is a **hosted connector** — there is nothing to install on your machine.
- To get access: ask **Li** (li@ecomnorth.com) to share the Master Leads DB connector with your claude.ai account. You add it under **claude.ai → Settings → Connectors**, then sign in when prompted.
- Full details: [`docs/master-leads-db.md`](docs/master-leads-db.md)

### 3. GetSales connector
Our outreach-sending platform (LinkedIn + email sequences).

- Also a hosted connector — add it in **claude.ai → Settings → Connectors** (ask Li for the connector if you don't see it).
- You need a seat on the GetSales team (**Nic's Team**) — ask Nic or Li.

### 4. NotebookLM skill (for the cold-email best-practices stage)
- Copy the `notebooklm` skill folder into `~/.claude/skills/` (ask Li for it — it's a separate skill, not in this repo).
- First use requires a one-time Google login: a browser window opens, you sign in with your Ecom North Google account.
- If you skip this, the tool still works — it just writes campaign copy from Claude's own judgment instead of our curated notebook, and tells you it did so.

---

## Installing the skill (2 minutes)

**Option A — with git:**
```bash
git clone https://github.com/ecomnorthteam/ecn-outreach-skill.git
cp -r ecn-outreach-skill/skills/ecn-outreach ~/.claude/skills/
```

**Option B — no git:** click the green **Code** button on this page → **Download ZIP** → unzip → copy the `skills/ecn-outreach` folder into the `.claude/skills` folder in your home directory (on Mac: press `Cmd+Shift+G` in Finder and type `~/.claude/skills`).

**Verify:** open Claude Code, type `pull some ecom leads` — Claude should mention the ECN Outreach pipeline and start asking intake questions.

---

## How to run a pull (examples)

Just type what you want, in plain English:

> "Pull 100 seven-figure beauty brands in Canada and build a Toronto Summit invite campaign, email + LinkedIn"

> "Find 50 CPG founders in Singapore for the August summit, email only"

> "Get me 30 logistics companies that might sponsor our LA founder dinner"

Claude will ask anything it still needs (lead count, revenue band, sender profile, etc.), then walk the pipeline with a checkpoint at each stage.

**Choosing the sender:** when it asks which GetSales sender profile to use, you can pick from the list it shows or just type the person's name (e.g. "use Coeli").

---

## What it costs per run

| Step | Cost |
|---|---|
| StoreLeads store discovery | **Free** (team subscription) |
| Apollo company/contact lookups | ~$0.05 per company revealed |
| Email verification (Emailable) | ~$0.001 per email |
| NotebookLM, GetSales build | Free |

A typical 100-lead pull costs **a few dollars**. Claude checks the team's daily budget before paid steps and warns you first.

---

## Safety rules (built in — you don't have to do anything)

1. **Campaigns are never started by the tool.** Every flow is created OFF. A human reviews and clicks Start in GetSales.
2. **Risky emails are never contacted.** Catch-all / low-confidence addresses go to a separate file for you to decide on manually.
3. **No double-contacting.** Every pull checks our shared outreach history (90-day window by default) and logs what it enrolls, so the next person's pull automatically skips those companies.
4. **It asks before spending.** Paid lookups and big batches get confirmed with you first.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| "Team selection required" from GetSales | Normal — Claude selects Nic's Team automatically. If you're on multiple teams it will ask. |
| Sender shows `email:error` when campaign is built | That sender's mailbox is disconnected in GetSales. Reconnect it there (Settings → Email accounts), or run LinkedIn-only. |
| NotebookLM asks for login / fails | Run the one-time Google login (Claude will offer it), or tell Claude "proceed without the notebook". |
| "I don't see the Master Leads DB tools" | The connector isn't added to your claude.ai account — see Prerequisites #2. |
| Very few leads come back | Normal for narrow filters. Claude will show you exactly which filter is constraining and offer to relax it. |

Anything else: ask Li, or just tell Claude what went wrong — the skill is built to explain itself.

---

## What's in this repo

```
skills/ecn-outreach/          the skill itself (copy this to ~/.claude/skills/)
  SKILL.md                    pipeline orchestrator
  references/stage-1..5.md    detailed playbooks per stage (incl. field-tested API gotchas)
  evals/evals.json            behavioral test cases
docs/master-leads-db.md       Master Leads DB connector — what it is, how to get access
docs/EVALS.md                 evaluation results (19/19 with skill vs 8/19 baseline)
```

---

## Privacy note

This repository is **private** — visible only to invited Ecom North team members, and not indexed by search engines. Keep it that way: it references our internal tooling and processes. Don't fork it to a public account.
