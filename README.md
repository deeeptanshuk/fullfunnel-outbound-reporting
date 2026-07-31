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
| **AI analysis** | Claude Sonnet 4.6 analyses each campaign individually, then writes a combined executive readout |
| **Slack output** | Posts a structured Slack message with reply highlights, key insights, and a CSV attachment |
| **Supabase write** | Saves a weekly snapshot per campaign for historical trend tracking |

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

Required tables:
- `client_configs` — one row per client, holds API keys and Slack channel
- `campaign_snapshots` — weekly metrics written after each run
- `campaign_change_logs` — change audit trail, used by the AI agents
- `campaign_messaging_kb` — *(planned)* messaging knowledge base

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

The AI agents check `campaign_change_logs` before recommending any optimization. If a matching change was logged within the last 14 days, the recommendation is suppressed and replaced with a note indicating the change is in-flight. Change logs are now **filtered per campaign** (not just per client) — see [CHANGELOG.md](CHANGELOG.md).

---

## Known Gaps & Roadmap

See [docs/GAP_TRACKING.md](docs/GAP_TRACKING.md) for full detail.

| # | Gap | Status |
|---|-----|--------|
| 1 | Change logs not broken out by campaign | ✅ Implemented (v1.1) |
| 2 | No unified cross-platform lead withdrawal | 🔲 Planned |
| 3 | No messaging knowledge base | 🔲 Planned |

---

## Friday Reminder

A separate branch of the workflow fires every **Friday at 4 PM** and sends a DM to the configured Slack user summarising which clients will receive Monday reports. This prompts analysts to log any strategic context before the automated run.

---

## License

Internal use — FullFunnel.
