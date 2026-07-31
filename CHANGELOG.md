# Changelog

All notable changes to this workflow are documented here.

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
