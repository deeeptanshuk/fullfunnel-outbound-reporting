# Supabase Schema

All tables live in the default `public` schema.

> **2026-08-01 note:** the sections below for `client_configs`, `campaign_snapshots`, `campaign_change_logs`, and `campaign_leads` were verified directly against the live database via `information_schema` queries. Where the real schema differed from what this file previously documented (notably `campaign_change_logs`), the real schema wins and is documented as-is below — these tables already exist, so there's no SQL to run for them. `campaign_messaging_kb` does **not** exist yet and still needs its CREATE TABLE run.

---

## `client_configs`

One row per client. Holds API keys, Slack destination, and the active flag consumed by the workflow loop.

```sql
create table public.client_configs (
  id                  uuid primary key default gen_random_uuid(),
  client_name         text not null unique,
  instantly_api_key   text,
  heyreach_api_key    text,
  slack_channel       text not null,  -- Slack channel ID (e.g. C01234ABCDE)
  is_active           boolean not null default true,
  created_at          timestamptz not null default now(),
  updated_at          timestamptz not null default now()
);

-- Keep updated_at current automatically
create or replace function public.set_updated_at()
returns trigger language plpgsql as $$
begin
  new.updated_at = now();
  return new;
end;
$$;

create trigger trg_client_configs_updated_at
  before update on public.client_configs
  for each row execute function public.set_updated_at();
```

**Notes:**
- Either `instantly_api_key` or `heyreach_api_key` may be null — the workflow skips that platform gracefully
- `slack_channel` must be a Slack **channel ID** (starts with `C` or `G`), not a channel name

---

## `campaign_snapshots`

Weekly performance snapshot written after each report run. Used by AI agents for previous-window comparison and trend tracking. **Verified live 2026-08-01 — matches this documentation.**

```sql
create table public.campaign_snapshots (
  id                                  uuid primary key default gen_random_uuid(),
  client_name                         text not null,
  campaign_id                         text not null,
  campaign_name                       text,
  platform                            text not null check (platform in ('instantly', 'heyreach')),
  reporting_window_start              date not null,
  reporting_window_end                date not null,

  -- Weekly window metrics
  sent_count                          integer,
  reply_count                         integer,
  open_count                          integer,
  bounce_count                        integer,
  connection_acceptance_count         integer,  -- HeyReach only

  -- Lifetime totals (as of report run)
  lifetime_sent_count                 integer,
  lifetime_reply_count                integer,
  lifetime_open_count                 integer,
  lifetime_bounce_count               integer,
  lifetime_connection_acceptance_count integer,  -- HeyReach only

  -- AI output
  verdict                             text,
  recommendations                     jsonb,
  raw_ai_insights                     text,

  created_at                          timestamptz not null default now(),

  unique (campaign_id, reporting_window_start, reporting_window_end)
);

create index on public.campaign_snapshots (client_name, reporting_window_end desc);
create index on public.campaign_snapshots (campaign_id, platform);
```

**Notes:**
- The unique constraint on `(campaign_id, reporting_window_start, reporting_window_end)` makes upserts idempotent — re-running the workflow for the same week overwrites cleanly
- `platform` distinguishes Instantly email snapshots from HeyReach LinkedIn snapshots for the same campaign name
- **Important asymmetry:** for Instantly rows, `campaign_id` actually holds the campaign **name** — the real Instantly numeric campaign ID never survives the AI output schema upstream (`campaignOverview` has no `id` field), so `Prepare Instantly Snapshot` falls back to name. HeyReach rows use the real numeric HeyReach campaign ID. This same convention is what `campaign_change_logs.campaign_id` uses too (see below), and what Gap 1's filtering matches against.

---

## `campaign_change_logs`

Analyst-logged changes to campaigns. Used by AI agents to suppress recommendations for in-flight optimisations.

**⚠️ Real schema (verified live 2026-08-01) — differs from earlier documentation in this file.** There is **no `campaign_name` column**. The real columns are:

```
id                   uuid              not null
client_name          character varying not null
campaign_id          character varying not null
platform             character varying not null
change_category      character varying not null
change_description   text              not null
actioned_by          character varying not null
actioned_at          timestamp with time zone  not null
```

(No separate `created_at` — `actioned_at` serves that purpose.) `campaign_id` and `platform` are both required on every row.

**Client-wide notes convention:** `platform = 'general'` (used by the Slack Command Handler's bulk "Campaign Context" flow) marks a row as applying to every campaign for that client, rather than one specific campaign+platform pair.

**Matching behaviour (as of v1.4):** the workflow matches logs to the campaign currently being reported on via `(campaign_id, platform)` — exact match, not fuzzy — with the `platform = 'general'` rows passed through to every campaign. See [GAP_TRACKING.md](GAP_TRACKING.md#gap-1--change-logs-not-broken-out-by-campaign) for the full history of how this was discovered and corrected; an earlier version of this fix (and this doc) assumed a `campaign_name` column that never existed, which meant the fix silently didn't work in production for several days.

**Common `change_category` values:**

| Category | What it covers | AI deduplication trigger |
|----------|---------------|--------------------------|
| `Subject Line` | Subject line rewrites or A/B tests | Suppresses "Refresh Subject Lines" recommendation |
| `Body Copy` | Message body or CTA changes | Suppresses "CTA & Body Copy A/B Test" recommendation |
| `ICP Target` | Audience or list changes | Suppresses "ICP Refinement" recommendation |
| `Sender Account` | Adding/removing sender accounts | Suppresses "Profile Optimisation" recommendation |
| `Campaign Context` | Strategic notes for the AI (e.g. "paused for list cleaning") | Appears in report context block |
| `Sequence` | Step order or timing changes | Suppresses "Sequence Timing Audit" recommendation |
| `Cross-Platform Withdrawal (Dry Run)` | Auto-logged by the workflow (v1.3/v1.4) when a reply on one platform matches a lead still active on the other — no live withdrawal happens yet, see [GAP_TRACKING.md](GAP_TRACKING.md#gap-2--no-unified-cross-platform-lead-withdrawal) | None — informational only |

---

## `campaign_leads`

**Discovered 2026-08-01, not created by either n8n workflow.** An independently-synced canonical lead registry — `last_synced_at` shows it's kept current by some external process (likely Clay or a dedicated sync job, given the original problem doc's mention of Clay and the presence of `external_lead_id`). Neither the reporting workflow nor the Slack Command Handler writes to this table; it's documented here purely as a **read** dependency for Gap 2.

**Real schema (verified live):**

```
id                    bigint            not null
client_name           text              not null
platform              text              not null
campaign_id           text              not null
campaign_name         text              (nullable)
external_lead_id      text              not null
email                 text              (nullable)
first_name            text              (nullable)
last_name             text              (nullable)
company_name          text              (nullable)
company_domain        text              (nullable)
linkedin_profile_url  text              (nullable)
lead_status           text              (nullable)
last_synced_at        timestamp with time zone  not null
created_at            timestamp with time zone  not null
```

**Observed from live sample data:**
- Instantly rows: `email` and `company_domain` populated; `linkedin_profile_url` always null
- HeyReach rows: `linkedin_profile_url` always populated; `email` populated on most (not all) sampled rows; `company_domain` not populated
- `lead_status` uses each platform's own native vocabulary — HeyReach: `Finished` / `InSequence` / `Pending` / `Failed` / `Excluded`; Instantly: numeric codes (`1`/`3`/`-1`/`-2`/etc., roughly Active/Completed/Bounced/Unsubscribed)
- `campaign_id` for Instantly rows here is a real UUID (Instantly's actual campaign ID) — **this is a different convention than `campaign_snapshots`/`campaign_change_logs`**, which store the campaign *name* as `campaign_id` for Instantly (see the asymmetry note under `campaign_snapshots` above). Don't assume `campaign_leads.campaign_id` and `campaign_change_logs.campaign_id` are joinable for Instantly rows — they usually aren't.

**Used by:** `Cross-Platform Withdrawal Check` (Gap 2) as the primary, high-confidence match source — an Instantly replier's email is checked against HeyReach rows here (and vice versa via LinkedIn URL → email lookup), with `lead_status` used to confirm the other platform's lead is still actively enrolled before flagging a dry-run withdrawal candidate.

---

## `campaign_messaging_kb`

Knowledge base of messaging approaches, outcomes, and learnings per campaign. Written automatically after every report run; read back at the start of every run so a new campaign for an existing client starts from prior context instead of zero.

**⚠️ Does not exist yet** — confirmed via a live `information_schema.tables` query on 2026-08-01. Run this before Gap 3 can actually work:

```sql
create table public.campaign_messaging_kb (
  id              uuid primary key default gen_random_uuid(),
  client_name     text not null,
  campaign_id     text not null,
  campaign_name   text,
  platform        text not null check (platform in ('instantly', 'heyreach')),
  icp_segment     text,       -- not yet populated by the workflow; reserved for future ICP tagging
  approach        text,       -- primary signal + what's working / areas to watch, derived from that week's AI output
  outcome         text,       -- that week's executive summary
  verdict         text,       -- AI verdict at the time (e.g. 'Strong Performance', 'Low Reply Rate')
  week_of         date not null,
  created_at      timestamptz not null default now(),

  unique (campaign_id, week_of)
);

create index on public.campaign_messaging_kb (client_name, week_of desc);
```

**Notes:**
- The unique constraint on `(campaign_id, week_of)` makes writes idempotent, same pattern as `campaign_snapshots`
- The workflow queries this table **per client, not per campaign** — up to the 15 most recent entries across all of that client's campaigns and platforms — so a new campaign can learn from what worked (or didn't) elsewhere for the same client
- `icp_segment` is reserved for when ICP tagging is added to `client_configs` or `campaign_change_logs`; currently always null
- `approach` and `outcome` are derived directly from the same AI agent output used for `campaign_snapshots.recommendations` / `raw_ai_insights` — no extra AI call is made to populate this table

---

## RLS (Row Level Security)

The workflow uses the **service role key**, which bypasses RLS. If you expose any of these tables to end users directly, enable RLS and add appropriate policies.

```sql
alter table public.client_configs enable row level security;
alter table public.campaign_snapshots enable row level security;
alter table public.campaign_change_logs enable row level security;
alter table public.campaign_messaging_kb enable row level security;
-- campaign_leads is owned by an external sync process — coordinate RLS changes with whatever manages it
```
