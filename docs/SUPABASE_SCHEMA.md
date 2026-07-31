# Supabase Schema

All tables live in the default `public` schema. Run these SQL statements in the Supabase SQL editor.

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

Weekly performance snapshot written after each report run. Used by AI agents for previous-window comparison and trend tracking.

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

---

## `campaign_change_logs`

Analyst-logged changes to campaigns. Used by AI agents to suppress recommendations for in-flight optimisations.

```sql
create table public.campaign_change_logs (
  id                  uuid primary key default gen_random_uuid(),
  client_name         text not null,
  campaign_name       text,          -- null = applies to all campaigns for this client
  change_category     text not null, -- e.g. 'Subject Line', 'Body Copy', 'ICP Target', 'Campaign Context'
  change_description  text not null,
  actioned_at         timestamptz not null default now(),
  actioned_by         text,          -- analyst name or identifier
  created_at          timestamptz not null default now()
);

create index on public.campaign_change_logs (client_name, actioned_at desc);
create index on public.campaign_change_logs (client_name, campaign_name, actioned_at desc);
```

**`campaign_name` filtering behaviour (as of v1.1):**
- Rows **with** `campaign_name` set are filtered to matching campaigns only (fuzzy match)
- Rows **without** `campaign_name` (null) are treated as client-wide and included for all campaigns

**Common `change_category` values:**

| Category | What it covers | AI deduplication trigger |
|----------|---------------|--------------------------|
| `Subject Line` | Subject line rewrites or A/B tests | Suppresses "Refresh Subject Lines" recommendation |
| `Body Copy` | Message body or CTA changes | Suppresses "CTA & Body Copy A/B Test" recommendation |
| `ICP Target` | Audience or list changes | Suppresses "ICP Refinement" recommendation |
| `Sender Account` | Adding/removing sender accounts | Suppresses "Profile Optimisation" recommendation |
| `Campaign Context` | Strategic notes for the AI (e.g. "paused for list cleaning") | Appears in report context block |
| `Sequence` | Step order or timing changes | Suppresses "Sequence Timing Audit" recommendation |
| `Cross-Platform Withdrawal (Dry Run)` | Auto-logged by the workflow (v1.3) when a reply on one platform fuzzy-matches a lead on the other — no live withdrawal happens yet, see [GAP_TRACKING.md](GAP_TRACKING.md#gap-2--no-unified-cross-platform-lead-withdrawal) | None — informational only |

---

## `campaign_messaging_kb`

Knowledge base of messaging approaches, outcomes, and learnings per campaign. Written automatically after every report run; read back at the start of every run so a new campaign for an existing client starts from prior context instead of zero.

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
```
