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

---

## `campaign_messaging_kb` *(Planned — Gap 3)*

Knowledge base of messaging approaches, outcomes, and learnings per campaign. Not yet implemented.

```sql
-- Planned schema — not active
create table public.campaign_messaging_kb (
  id              uuid primary key default gen_random_uuid(),
  client_name     text not null,
  campaign_name   text,
  platform        text,
  icp_segment     text,
  approach        text,      -- what was tried
  outcome         text,      -- what happened
  verdict         text,      -- AI verdict at the time
  week_of         date,
  created_at      timestamptz not null default now()
);
```

---

## RLS (Row Level Security)

The workflow uses the **service role key**, which bypasses RLS. If you expose any of these tables to end users directly, enable RLS and add appropriate policies.

```sql
alter table public.client_configs enable row level security;
alter table public.campaign_snapshots enable row level security;
alter table public.campaign_change_logs enable row level security;
```
