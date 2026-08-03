# FullFunnel Outbound Reporting

An n8n workflow that automatically reports on outbound performance every week, for every client, across Instantly (email) and HeyReach (LinkedIn).

## What it does

Every Monday, for each active client, it pulls that week's campaign data from Instantly and HeyReach, has Claude write an analysis, and posts it to the client's Slack channel with a CSV attached. It also remembers what's been tried before (past messaging, logged changes) so each week's analysis builds on the last, instead of starting from zero.

## Status: the 3 gaps this was built to fix

| Gap | What it means | Status |
|---|---|---|
| **1. Change logs weren't per-campaign** | A note logged for one campaign could wrongly show up in a different campaign's report | ✅ Fixed and tested live |
| **2. No cross-platform withdrawal** | A lead who replied on one channel kept getting messaged on the other | 🟡 Detects and flags for manual review — doesn't pause leads automatically yet |
| **3. No messaging history** | Every new campaign started from zero, no memory of what worked before | ✅ Fixed and tested live |

Full detail on each, including bugs found during testing and what's still open, is in [docs/GAP_TRACKING.md](docs/GAP_TRACKING.md).

## Setup

1. Create the required Supabase tables — see [docs/SUPABASE_SCHEMA.md](docs/SUPABASE_SCHEMA.md)
2. Import `workflows/v1-single-engine-scheduled-outbound-slack-pulse.json` into n8n
3. Add credentials: an Anthropic API key, a Slack bot token, and the Supabase service role key (replace the `YOUR_SUPABASE_SERVICE_ROLE_KEY` placeholders)
4. Add each client as a row in `client_configs` (client name, Instantly/HeyReach API keys, Slack channel)
5. Activate the workflow

Each client's own Instantly and HeyReach API keys live in `client_configs`, not in n8n — one workflow serves every client.

## How it decides what to say

The AI verdicts (Early Stage, Low Reply Rate, Strong Performance, etc.) and the full node-by-node flow are documented in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## More detail

- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — how the workflow is built, node by node
- [docs/SUPABASE_SCHEMA.md](docs/SUPABASE_SCHEMA.md) — every table and column this depends on
- [docs/GAP_TRACKING.md](docs/GAP_TRACKING.md) — each gap's problem, fix, and current status
- [CHANGELOG.md](CHANGELOG.md) — version history

## License

Internal use — FullFunnel.
