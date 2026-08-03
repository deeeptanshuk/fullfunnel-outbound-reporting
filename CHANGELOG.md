# Changelog

All notable changes to this workflow are documented here.

---

## [v1.5] — 2026-08-04

### Fixed — HeyReach `campaign_id` fallback stored the campaign name instead of the real numeric ID

`Prepare HeyReach Snapshot`'s `campaign_id` fallback chain (`item.campaignConfig?.id || item.campaignOverview?.campaignId || Number($json.matchedCampaignId) || item.campaignName`) always fell through to the campaign name, because `$json.matchedCampaignId` does not survive through the AI Agent node's structured output (`Number(undefined)` is `NaN`, which is falsy). Confirmed via live testing against two different clients (PLANHUB, CONSTELLATION) — every historical `campaign_snapshots` row for HeyReach had the name stored where the numeric ID should be.

Fixed by looking up the real numeric ID directly from `Match Heyreach Campaign Name - Exist or not` (the node that actually captures it from HeyReach's live API, before the AI agent runs), matched by campaign name. Verified live post-fix: a fresh HeyReach snapshot row now correctly stores the real numeric ID (e.g. `516758`) instead of the name.

**Why this matters:** the Slack Command Handler's campaign dropdown builds its value directly from `campaign_snapshots.campaign_id`. With the name stored there instead of the ID, any change a person logged in Slack against a HeyReach campaign would carry the wrong identifier and could silently fail Gap 1's per-campaign filtering — the exact class of bug Gap 1 was built to prevent, just introduced from a different angle.

### Fixed — HeyReach change logs could be permanently lost during the ID migration

The fix above (real numeric HeyReach `campaign_id`) created a gap: the Slack Command Handler's dropdown builds its value from whatever's currently in `campaign_snapshots.campaign_id`, so a change logged via Slack *before* a campaign's snapshot refreshed under the new numeric ID would be saved under the old campaign name instead. Unlike `campaign_snapshots` (rewritten every week), `campaign_change_logs` is write-once — so that note would never match the report filter again, silently, forever.

Fixed by having `Merge Heyreach Nodes Data`'s `filterLogsForCampaign` accept either the real numeric ID or the campaign's current name, so a note lands correctly regardless of which era it was logged in. No manual data cleanup needed.

### Fixed — `previousSnapshot` (week-over-week trend comparison) never actually populated, for any client, on either platform

Two separate bugs compounded to make this feature a no-op since it was first built:

1. **`Normalize Client History`** had the same parsing bug later found in `Normalize Messaging KB` (see v1.2's KB fix below) — it assumed the first fetched Supabase row already contained an array, when Supabase actually returns one n8n item per row. Result: `Normalize Client History` always emitted `{ is_empty_history: true }`, so `previousSnapshot` was always `null` on both platforms, regardless of how much real history existed. Fixed with the same "map over all returned items" pattern as the KB fix.

2. **Instantly-specific identifier mismatch**, found after fixing #1: `Merge Report Data`'s `previousSnapshot` lookup matched `campaign_snapshots.campaign_id` against the real Instantly campaign ID (`matchData.matchedCampaignId`), but `Prepare Instantly Snapshot` stores the campaign **name** there (Instantly's existing, correct convention — see Gap 1 below). Since the lookup searched for the wrong identifier type, it could never match, even with #1 fixed. Fixed by matching against `targetCampaignName` instead, consistent with how change-log filtering already works in the same function. HeyReach's equivalent lookup already used the real numeric ID correctly and needed no change, since HeyReach's snapshot storage convention is the real ID (see the v1.5 fix above).

Both fixes verified via live execution data inspection; full end-to-end proof (a real second week of history for the same campaign) is still pending, since the currently active test campaigns are too new to have a genuinely prior week to match against.

### Fixed — Change-log entries reaching the AI were duplicated up to 11x

`Fetch Campaign Change Logs` sits downstream of `Fetch Client History` in the node graph. Since `Fetch Client History` can return multiple items (one per historical snapshot row), the change-log fetch — which is client-wide, not per-item — re-ran redundantly once per upstream item, producing many duplicate copies of the same rows. Confirmed via live execution: 11 copies of a single change-log row reached the AI agent's context in one run. Per-campaign filtering correctness itself was unaffected (verified no cross-campaign leakage), this was pure wasted duplicate content, and wasted tokens sent to Anthropic on every run.

Fixed by deduplicating by row `id` immediately after reading `Fetch Campaign Change Logs`' output, in both `Merge Report Data` and `Merge Heyreach Nodes Data`. Verified live post-fix: change-log count per campaign dropped from 11 to 1.

### Testing — Gap 1 re-confirmed live against a second client

Re-ran Gap 1 verification against CONSTELLATION (in addition to the original PLANHUB testing in v1.4), covering both an Instantly campaign (name-keyed) and a HeyReach campaign (numeric-ID-keyed). Confirmed no cross-campaign leakage in either case, and confirmed the existing asymmetry (Instantly uses the campaign name as its real, working identifier throughout; HeyReach uses the real numeric ID) is a deliberate, working design choice, not a bug — see Gap 1 notes in `docs/GAP_TRACKING.md`.

### In progress — production cutover

Reverted all testing-only shortcuts added during this round of testing: the hardcoded test Slack channel across all 6 Slack-output nodes (restored to each client's real channel, pulled dynamically from `client_configs`), and the temporary single-client filter on `Fetch Client Configs` (restored to processing every active client). The workflow remains **inactive** pending two coordination items with the separate production n8n account this replaces: (1) confirming the real production Slack bot credentials vs. continuing with the dedicated test bot, and (2) sequencing deactivation of the old workflow with activation of this one, to avoid a window where both could run.

---

## [v1.4] — 2026-08-01

### Fixed — Gap 1 change-log filtering was matching a nonexistent column

Live inspection of Supabase (`information_schema.columns`) found that `campaign_change_logs` has no `campaign_name` column at all — only `campaign_id` and `platform` (both `NOT NULL`). The v1.1/v1.2 `filterLogsByCampaign()` filtered on `campaign_name`, so every log's match check silently evaluated to "client-wide" against the real table — **Gap 1 was never actually fixed in production**, despite looking correct in the code.

Replaced with `filterLogsForCampaign(logs, campaignIdValue, platform)` in both `Merge Report Data` and `Merge Heyreach Nodes Data`, matching on `(campaign_id, platform)` with `platform = 'general'` preserved as the client-wide passthrough (the existing convention used by the Slack Command Handler's bulk-context flow). Instantly matches against campaign name (its real ID doesn't survive the AI schema upstream, so that's what's actually stored); HeyReach matches against its real numeric campaign ID. See `docs/GAP_TRACKING.md` for the full trace.

**Gap status:** Gap 1 — now actually resolved ✅ (previous "resolved" status was against the wrong schema)

### Changed — Gap 2 upgraded to real identifier matching via `campaign_leads`

Discovered a third Supabase table, `campaign_leads` — populated independently of both n8n workflows (likely Clay or a dedicated sync job), with `email` on Instantly rows and both `email` (usually) and `linkedin_profile_url` on HeyReach rows, keyed by `client_name`/`platform`/`campaign_id`/`lead_status`.

**Nodes added:** `Fetch Campaign Leads`, `Normalize Campaign Leads` — wired into both merge barriers (`Merge` 7→8 inputs, `Merge all heyreach nodes data` 6→7 inputs).

`Cross-Platform Withdrawal Check` rewritten to try an **exact email match** against `campaign_leads` first (checking the other platform's `lead_status` for "still active"), falling back to the v1.3 fuzzy name+company match only when no email match exists. Every match now carries a `confidence` label (`high (email match)` / `medium (name+company fuzzy match)`) in both the Slack notification and the Supabase log entry.

`Log Dry-Run Withdrawal to Supabase` corrected to send `campaign_id`/`platform` (the real columns) instead of the nonexistent `campaign_name`.

**Still open:** live withdrawal APIs are not called; the dry-run Slack notification still posts to each client's own report channel rather than a dedicated test channel (pending a channel ID).

### Fixed — Gap 3's table doesn't exist yet

A live `information_schema.tables` query confirmed `campaign_messaging_kb` was never created in Supabase. The workflow's read/write nodes have been correct since v1.2, but the table itself needs the CREATE TABLE migration run in the Supabase SQL editor (SQL in `docs/SUPABASE_SCHEMA.md` and `docs/GAP_TRACKING.md`) before Gap 3 actually functions.

---

## [v1.3] — 2026-08-01

### Added — Cross-platform withdrawal, dry-run only (Gap 2)

**Nodes added:** `Cross-Platform Withdrawal Check`, `Log Dry-Run Withdrawal to Supabase`, `Notify Dry-Run Withdrawal Match`
**Nodes extended:** `Accumulate Leads` (now also tracks a lightweight full lead roster — email/name/company for every lead, not just repliers), `Merge Heyreach Nodes Data` (now exposes `leadsAnalysis.connectedLeads` as a full array, not just a count)

- Every report run now checks whether an Instantly replier is still on the client's HeyReach roster (or vice versa), matched by normalised name + company
- No shared identifier (email / LinkedIn URL) exists between the two platforms' currently-collected data, so this is a fuzzy, medium-confidence match — not the exact-identifier match originally proposed
- Matches are logged to `campaign_change_logs` (`change_category: 'Cross-Platform Withdrawal (Dry Run)'`) and posted to the client's Slack channel, clearly labelled DRY RUN
- **No withdrawal API is called** — this detects and reports only. Tele-prospecting is out of scope (no tool/API identified yet)

**Why this matters:** the problem doc's core ask — stop messaging someone who already replied elsewhere — needs verification against real weeks of data before it's safe to wire to a live `StopLeadInCampaign` / unsubscribe call. This gives the team visibility into what the matcher would do without touching a single live lead.

**Gap status:** Gap 2 — dry-run implemented, live withdrawal still to come 🟡

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
