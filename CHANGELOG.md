# Changelog

All notable changes to this workflow are documented here.

---

## [v1.2] — 2026-07-31

### Added — Messaging knowledge base (Gap 3)

**Nodes added:** `Fetch Messaging KB`, `Normalize Messaging KB`, `Prepare Instantly KB Entry`, `Write Instantly KB Entry to Supabase`, `Prepare HeyReach KB Entry`, `Write HeyReach KB Entry to Supabase`

**New table:** `campaign_messaging_kb` (see [SUPABASE_SCHEMA.md](docs/SUPABASE_SCHEMA.md)) — run the migration before activating.

- After every report, each campaign's `approach`, `outcome`, and `verdict` are written to `campaign_messaging_kb`, derived from that week's AI output (no extra AI call)
- Before generating a report, the workflow now reads the client's 15 most recent KB entries across **all** of their campaigns and platforms and passes them to both AI agents as `reportData.priorMessagingContext`
- Both AI agent prompts now instruct the model to avoid recommending approaches already shown to fail for this client, and to flag cross-campaign/cross-platform patterns

**Why this matters:** Previously every new campaign for an existing client started from zero — no visibility into what messaging had already been tried. Now a client's other campaigns (and other platforms) inform each new one's analysis from week one.

**Gap closed:** Gap 3 — No messaging knowledge base ✅

### Fixed — Gap 1 live/GitHub drift

The live n8n workflow predated the v1.1 `filterLogsByCampaign` fix below — it had never been applied to the running workflow, only committed here. Backported while touching `Merge Report Data` and `Merge Heyreach Nodes Data` for the KB work above; also extended the fix to the `Campaign Context` lookup, which previously scanned the unfiltered change-log list.

---

## [v1.1] — 2026-07-31

### Changed — Campaign-level change log filtering

**Nodes affected:** `Merge Report Data`, `Merge Heyreach Nodes Data`

Added `filterLogsByCampaign(logs, campaignName)` helper in both nodes. Change logs fetched from `campaign_change_logs` are now filtered to the current campaign before being passed to the AI agents.

- Logs **without** a `campaign_name` value are treated as client-wide and flow through to all campaigns
- Fuzzy name matching (normalised lowercase, alphanumeric only, substring both directions) handles naming inconsistencies like bracket prefixes or platform suffixes
- `reportData.changeLogs` now contains only the filtered set, so the AI prompt receives only relevant history

**Why this matters:** Previously, when a client had multiple active campaigns, AI agents saw change logs from all campaigns. This caused incorrect deduplication — e.g. a subject line change on Campaign A would suppress a subject line recommendation for Campaign B.

**Gap closed:** Gap 1 — Change logs not broken out by campaign ✅

---

## [v1.0] — 2025-12-13

### Initial release

- Auto-discovers all active Instantly and HeyReach campaigns per client (no hardcoded campaign names required)
- Weekly cron trigger (Mon 9AM) + on-demand Webhook trigger with optional `client_name` filter
- Friday 4PM reminder DM listing active clients for the upcoming Monday run
- AI analysis via Claude Sonnet 4.6 with adaptive thinking (campaign agents) and standard mode (combined readout)
- Structured Slack pulse per client: reply highlights, What's Working / Areas to Watch, executive readout paragraph
- CSV attachment with weekly + lifetime metrics side by side: `"42 (187)"`
- Snapshot upserts to Supabase `campaign_snapshots` — idempotent on `(campaign_id, reporting_window_start, reporting_window_end)`
- Change log deduplication — AI suppresses recommendations matching in-flight changes logged within 14 days
- Graceful fallback for single-platform clients (Instantly-only or HeyReach-only)
- Paginated lead fetch — up to 5,000 leads, memory-safe accumulator (counts + repliers only)
- OOO and auto-reply filtering on Instantly received emails
- Sentiment classification on HeyReach inbox replies (positive / neutral / negative / informational)
