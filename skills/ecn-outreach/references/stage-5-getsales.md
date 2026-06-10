# Stage 5 — GetSales Campaign Build

All via the `getsales` MCP: `list_toolsets` → `get_toolset_tools({toolset, lite:true})` → `call_tool`. Relevant toolsets: `automations`, `crm`, `data-sources`, `email`.

**The safety contract: the flow is created in `off` status and stays off. NEVER call `control_flow({action:'start'})`. The human starts it in the GetSales UI.**

## Build order

### 0. Team selection

First mutation call may fail with "Team selection required" listing teams. Single team with active subscription → `call_tool({toolset:'account', tool_name:'select_team', tool_input:{team_id}})` without asking. Multiple → ask user.

### 1. Senders

`call_tool({toolset:'automations', tool_name:'manage_sender_profiles', tool_input:{action:'list'}})`. Present ALL senders as a compact name/timezone/label table (including disabled — flag those). The user may have already named the sender in their request or intake — match free-typed names against the list (first name alone is fine if unambiguous) and just confirm. Otherwise AskUserQuestion with the likeliest senders as options (match sender timezone to target geo) — the user can always type any name via Other. Don't guess channel readiness from the list — `create_flow` validates it and returns `health_issues[]` (e.g. `mailbox in non-active state (email:error)`) — surface those verbatim in the final handoff.

### 2. Structure

- **Email + LinkedIn**: `list_flow_templates` → match campaign playbook to closest T1-T7 (T5 = canonical multichannel: connect → accepted: 3 LI msgs / timeout: email → InMail → withdraw) → `get_flow_template({id})`. **Field-tested shortcut:** edit the copy directly inside the template's node payloads (nodes 3/6/7/8 LI texts, node 9 email subject+template, node 10 InMail) and pass the modified `nodes` + `first_common_node_id` + `template_id` to `create_flow` in one call — wiring stays canonical, no post-create `update_flow_step` round-trips needed. Do NOT alter ids/before/after/filters.
- **Email only**: build nodes from the playbook skeleton — chain of email-send nodes with `delay_in_seconds` between steps matching playbook cadence (days × 86400). Use `get_node_template({type})` for each node type's minimal valid payload; `list_node_types` if unsure of type strings. Consider a `rule_filter` `has_work_email` gate via `get_filter_snippet` if any enrolled lead might lack email.

### 3. Copy

1. `call_tool({toolset:'crm', tool_name:'list_template_variables'})` FIRST — the flat namespace of built-ins + custom fields. Only use variables that exist.
2. Write every step's subject + body from the Stage 4 playbook. Ecom North voice: founder-to-founder, direct, no corporate fluff. Personalization sources available from the pipeline: first_name, company name, niche/industry, platform (Shopify/Amazon), city (powerful for summit invites — "you're in Toronto, so is the summit").
3. If pipeline-derived fields (platform, niche, revenue band) should drive personalization, they must travel as custom fields on the leads at enrollment (step 5) AND exist in the template-variable namespace. `enroll_leads_in_flow` auto-creates unknown custom-field definitions — name them snake_case: `store_platform`, `niche`, `city`.
4. A/B test only if the user asked: `rule_ab_test` node, weights sum to 100.

### 4. Create flow

`create_flow` with: name (`ECN [BU] — [campaign] — [date]`), sender_profiles, nodes (or template_id), schedule. Flow lands in `off`.

- Relay `post_build_summary` and `graph_ascii` VERBATIM — don't paraphrase.
- Surface every `health_issues[]` / `sender_status[]` warning — these are the things that fail silently after the human clicks start.
- Schedule: default sender-schedule mode unless playbook/user wants a flow-level window (then `update_flow` with `use_sender_schedule:false` + `schedule.timeblocks`).

### 5. Enroll leads

MAIN bucket only. Risky bucket is never enrolled this run.

- **< 100 leads**: `enroll_leads_in_flow` — REQUIRES `list_uuid`: create one first via `call_tool({toolset:'crm', tool_name:'manage_lists', tool_input:{action:'create', name:'ECN <campaign> <date>'}})`. Per lead include `first_name` (required for email-only leads), `work_email`, `last_name`, `company_name`, `linkedin_id` (full LinkedIn URL works — universal parser) when channels include LinkedIn, and `custom_fields` for personalization (auto-creates definitions, e.g. `store_platform`, `niche`, `summit_city`). For email-only campaigns pass `confirm_email_only_mode: true` when leads lack linkedin_id.
- **100-500**: either path; prefer `enroll_leads_in_flow` for validation.
- **> 500**: `bulk_import_to_flow` (CSV path) — build the CSV from the run scratch dir, provide explicit `mapping` (backend automapping is unreliable), answer its mandatory pre-flight (tag `ecn-<campaign>-<date>`, no force_list_move, tag-new-only) and confirm with the user.

The flow is off, so enrollment is dormant — still tell the user: "enrolling N leads; they activate when you start the flow."

Tagging: `enroll_leads_in_flow` drops tags silently — tag via a tag-node at flow start, or `manage_tags({action:'add_to_leads'})` post-enrollment with UUIDs from `get_flow_lead_status`.

### 6. Log outreach to Master DB

`log_outreach` (Master Leads DB): `business_unit`, `company_ids` (UUIDs from the master DB upserts — they're on the Stage 1/2 records), `channel` ('email' or log twice for email+linkedin), `external_system:'getsales'`, `campaign_name`, `status:'queued'`. This is what makes the next pull's dedup work — never skip it.

## Final handoff (exact format)

```
## Campaign ready — awaiting your start
Flow:     [name] — [flow_url from post_build_summary]
Status:   OFF (nothing sends until you start it)
Senders:  [list + any health warnings]
Enrolled: N leads (MAIN bucket)
Held:     N risky-email contacts → ./runs/<dir>/risky-contacts.json
Sequence: [graph_ascii]

→ Review copy + senders in GetSales, then click Start.
```

Then complete the Run report from SKILL.md.

## Known sharp edges

- `create_flow` runs 3 backend requests; if it returns partially-failed state, `get_flow` to inspect before retrying — don't blind-retry (duplicate flows).
- Don't change a lead's `linkedin_id` via enroll — delete + recreate is the safe path.
- `manage_flow_contact_sources({action:'set_filter'})` on a running flow auto-enrolls future matches — avoid in this pipeline; we enroll explicitly.
- Numeric-only linkedin_id (`/^\d{6,}$/`) = member ID, not username — fix before enrolling.
