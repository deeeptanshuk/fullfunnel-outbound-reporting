# FullFunnel Outbound Reporting — n8n Workflow

Automated weekly outbound reporting across **Instantly** (email) and **HeyReach** (LinkedIn). Runs every Monday at 9 AM, pulls live campaign data, analyses performance with Claude AI, and posts a structured Slack pulse with an attached CSV.

---

## What It Does

| Step | Description |
|------|-------------|
| **Trigger** | Monday 9 AM cron, Friday 4 PM reminder DM, or on-demand via Webhook |
| **Client loop** | Reads all active client configs from Supabase and processes them one by one |
| **Instantly (Email)** | Fetches active campaigns, analytics, replies, lead counts, and sender accounts |
| **HeyReach (LinkedIn)** | Fetches active campaigns, overall stats, inbox conversations, and LinkedIn accounts |
| **AI analysis** | Claude Sonnet 4.6 analyses each campaign individually — informed by this client's messaging knowledge base — then writes a combined executive readout |
| **Slack output** | Posts a structured Slack message with reply highlights, key insights, and a CSV attachment |
| **Supabase write** | Saves a weekly snapshot and a messaging knowledge base entry per campaign for historical trend tracking |

---

## Repository Layout

```
fullfunnel-outbound-reporting/
├── workflows/
│   └── v1-single-engine-scheduled-outbound-slack-pulse.json  # Import this into n8n
├── docs/
│   ├── ARCHITECTURE.md     # Node-by-node flow explanation
│   ├── SUPABASE_SCHEMA.md  # Required database tables and columns
│   └── GAP_TRACKING.md     # Known gaps and implementation status
├── CHANGELOG.md            # Version history
└── README.md               # This file
```

---

## Quick Start

### 1. Supabase — Create Required Tables

See [docs/SUPABASE_SCHEMA.md](docs/SUPABASE_SCHEMA.md) for the full SQL.

Required tables — **`campaign_messaging_kb` does not exist yet as of 2026-08-01; run its migration in [docs/SUPABASE_SCHEMA.md](docs/SUPABASE_SCHEMA.md) before Gap 3 will work:**
- `client_configs` — one row per client, holds API keys and Slack channel
- `campaign_snapshots` — weekly metrics written after each run
- `campaign_change_logs` — change audit trail, used by the AI agents (real columns are `campaign_id`/`platform`, not `campaign_name` — see [docs/SUPABASE_SCHEMA.md](docs/SUPABASE_SCHEMA.md))
- `campaign_messaging_kb` — messaging knowledge base, written after each run and read back before every report — **needs creating**
- `campaign_leads` — read-only dependency for Gap 2, synced by a process external to this workflow (not something you create here)

### 2. n8n — Import the Workflow

1. Open your n8n instance
2. Go to **Workflows → Import from File**
3. Select `workflows/v1-single-engine-scheduled-outbound-slack-pulse.json`
4. Set up credentials (see below)

### 3. Credentials Required

| Credential | Used By |
|-----------|---------|
| `anthropicApi` | All three AI agent nodes (Instantly agent, HeyReach agent, Combined Readout agent) |
| `slackApi` | Report message, CSV upload, campaign-not-found alerts |
| Supabase service role key | All Supabase HTTP nodes — hardcoded as `YOUR_SUPABASE_SERVICE_ROLE_KEY` (replace before activating) |

> **Note:** Instantly and HeyReach API keys are stored per-client in the `client_configs` Supabase table, not as n8n credentials. This allows multi-client operation with a single workflow.

### 4. client_configs Table — Minimum Row

```sql
INSERT INTO client_configs (
  client_name,
  instantly_api_key,
  heyreach_api_key,
  slack_channel,
  is_active
) VALUES (
  'Acme Corp',
  'inst_xxxx',
  'hr_xxxx',
  'C01234ABCDE',   -- Slack channel ID (not name)
  true
);
```

Either `instantly_api_key` or `heyreach_api_key` may be empty — the workflow skips the platform gracefully.

### 5. Activate

Enable the workflow in n8n. It will fire automatically on the cron schedule, or you can POST to the webhook URL to trigger a specific client on demand:

```bash
curl -X POST https://your-n8n-instance/webhook/9ec82870-3048-46eb-9d8d-ca40cacaabf8 \
  -H "Content-Type: application/json" \
  -d '{"client_name": "Acme Corp", "slack_channel": "C01234ABCDE"}'
```

---

## Reporting Window

The workflow always reports on **the previous full week** (Monday–Sunday), derived at runtime:

```
startDate = now minus 7 days → start of that week
endDate   = now minus 7 days → end of that week
```

---

## AI Verdict Rules

### Instantly (Email)

| Verdict | Condition |
|---------|-----------|
| Early Stage | Campaign age < 14 days |
| High Bounce Rate | Bounce rate > 5% |
| Low Open Rate | Open rate < 20% AND bounce ≤ 5% |
| Low Reply Rate | Open ≥ 20% AND reply rate < 1% |
| Strong Performance | Open ≥ 40% AND reply ≥ 3% |

### HeyReach (LinkedIn)

| Verdict | Condition |
|---------|-----------|
| Early Stage | Campaign age < 14 days |
| Low Acceptance | Connection acceptance rate < 15% |
| Low Reply Rate | Acceptance ≥ 15% AND message reply rate < 5% |
| Strong Performance | Acceptance ≥ 25% AND message reply rate ≥ 10% |

---

## Change Log Deduplication

The AI agents check `campaign_change_logs` before recommending any optimization. If a matching change was logged within the last 14 days, the recommendation is suppressed and replaced with a note indicating the change is in-flight. Change logs are filtered **per campaign** (not just per client) — see [CHANGELOG.md](CHANGELOG.md).

---

## Messaging Knowledge Base

Every report run writes a `campaign_messaging_kb` entry per campaign (approach, outcome, verdict) and reads back the client's 15 most recent entries across all campaigns and platforms before generating the next report. This is deliberately **not** filtered to the current campaign — unlike change logs, the goal is surfacing what worked (or didn't) elsewhere for the same client, so a brand-new campaign isn't starting from zero. See [CHANGELOG.md](CHANGELOG.md) and [docs/GAP_TRACKING.md](docs/GAP_TRACKING.md#gap-3--no-messaging-knowledge-base).

---

## Known Gaps & Roadmap

See [docs/GAP_TRACKING.md](docs/GAP_TRACKING.md) for full detail.

| # | Gap | Status |
|---|-----|--------|
| 1 | Change logs not broken out by campaign | ✅ Implemented (v1.4 — earlier "resolved" status in v1.1/v1.2 was against a schema that turned out to be wrong; see below) |
| 2 | No unified cross-platform lead withdrawal | 🟡 Dry-run only (v1.4) — detects & logs with confidence tiers, does not call withdrawal APIs yet |
| 3 | No messaging knowledge base | 🟡 Code implemented (v1.2) — **its Supabase table doesn't exist yet**, see Quick Start above |

---

## Cross-Platform Withdrawal (Dry Run)

Every report run checks whether anyone who replied on Instantly this week is still active on HeyReach for the same client (or vice versa). As of v1.4 this tries an **exact email match first** against `campaign_leads` — a canonical lead registry synced independently of this workflow that carries email on both platforms' rows (and LinkedIn URL on HeyReach's) — checking the other platform's status to confirm the lead is still actively enrolled. Only falls back to normalised name+company fuzzy matching when no email match exists. Every match is labelled with its confidence (`high` or `medium`), logged to `campaign_change_logs`, and posted to Slack labelled **DRY RUN** — no lead is actually stopped or unsubscribed yet. See [docs/GAP_TRACKING.md](docs/GAP_TRACKING.md#gap-2--no-unified-cross-platform-lead-withdrawal).

---

## Friday Reminder

A separate branch of the workflow fires every **Friday at 4 PM** and sends a DM to the configured Slack user summarising which clients will receive Monday reports. This prompts analysts to log any strategic context before the automated run.

---

## License

Internal use — FullFunnel.
