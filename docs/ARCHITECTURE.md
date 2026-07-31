# Architecture — Node-by-Node Flow

This document walks through every stage of the workflow in execution order.

---

## Trigger Layer

Three entry points converge on `Fetch Client Configs`:

| Node | Type | When |
|------|------|------|
| `Schedule Trigger (Mon 9AM)` | Cron `0 9 * * 1` | Every Monday |
| `Webhook` | HTTP POST | On-demand; body may include `client_name` and `slack_channel` to target a single client |
| `When clicking "Test workflow"` | Manual | n8n canvas test runs |

---

## Client Loop

```
Fetch Client Configs → Loop active clients → Workflow Configuration
```

**`Fetch Client Configs`** — GET `client_configs` from Supabase filtered by `is_active = true`.

**`Loop active clients`** — `splitInBatches` node (batch size 1). Iterates through every active client row. The loop exits when `noItemsLeft` is true.

**`Workflow Configuration`** — Set node that maps the client row fields to named variables used throughout:

| Variable | Source |
|----------|--------|
| `clientName` | `client_name` |
| `instantlyApiKey` | `instantly_api_key` |
| `heyReachApiKey` | `heyreach_api_key` |
| `slackChannel` | `slack_channel` (webhook body overrides table value if present) |
| `startDate` | Previous week Monday (`now - 7d → startOf('week')`) |
| `endDate` | Previous week Sunday (`now - 7d → endOf('week')`) |

**`Filter Webhook Client`** — When triggered via Webhook with a `client_name` in the body, skips clients that don't match. When triggered by cron or manual, passes all clients through.

---

## Platform Branching

After `Filter Webhook Client`, execution splits into two parallel tracks plus two shared fetches:

```
Filter Webhook Client
  ├─→ Check Instantly Key  (email track)
  ├─→ Check HeyReach Key   (LinkedIn track)
  ├─→ Fetch Client History (shared — feeds both tracks)
  └─→ Fetch Messaging KB   (shared — feeds both tracks)
```

### Shared History Fetch

```
Fetch Client History → Fetch Campaign Change Logs → Normalize Client History
Fetch Messaging KB → Normalize Messaging KB
```

**`Fetch Client History`** — Reads `campaign_snapshots` for this client ordered by `reporting_window_end desc`. Used by AI agents to surface previous-window comparison.

**`Fetch Campaign Change Logs`** — Reads `campaign_change_logs` for this client since `startDate`. Contains all analyst-logged changes (subject line edits, ICP changes, context notes, etc.).

**`Normalize Client History`** — Code node that parses the Supabase array response into n8n items, with a safe fallback for empty history.

**`Fetch Messaging KB`** — Reads up to the 15 most recent `campaign_messaging_kb` rows for this client, ordered by `week_of desc`. Deliberately queried **per client, not per campaign** — the goal is surfacing what worked (or didn't) on this client's other campaigns and platforms, not just this campaign's own history.

**`Normalize Messaging KB`** — Same parsing pattern as `Normalize Client History`, with an `is_empty_kb` fallback for clients with no KB history yet.

Normalized history and KB entries then feed into both the Instantly `Merge` node (inputs 5 and 6) and the HeyReach `Merge all heyreach nodes data` node (inputs 4 and 5).

---

## Instantly (Email) Track

### Campaign Discovery

```
Check Instantly Key → Get many campaigns → Match Instantly Campaign Name
                   ↘ [no key] Mock Instantly Output + Mock Instantly Lifetime
```

**`Check Instantly Key`** — If `instantlyApiKey` is empty, routes to mock nodes so the report still generates (LinkedIn-only clients).

**`Get many campaigns`** — GET `/api/v2/campaigns`. Returns all campaigns regardless of status.

**`Match Instantly Campaign Name - Exist or not`** — Code node. Filters to `status === 1` (active only). Outputs one item per active campaign with `matched: true`. Falls back to `matched: false` if none found.

**`Instantly Campaign Existence Check`** — IF node routing:
- `matched = true` → `Wait - Instantly API`
- `matched = false` → `Check If Instantly Target Is Null`

**`Check If Instantly Target Is Null`** — Distinguishes "no campaigns at all" (mock route) from "named campaign not found" (Slack alert route). Since v1 auto-discovers active campaigns, this handles the no-active-campaigns case.

**`Wait - Instantly API`** — 1-second pause to avoid rate limiting before fanning out.

### Data Collection (parallel after Wait)

All six branches fire simultaneously:

| Node | API Call | Purpose |
|------|----------|---------|
| `Get matched campaign data` | GET `/campaigns/{id}` | Full campaign config incl. sequences, sender list |
| `Get Campaign Analytics` | GET `/campaigns/analytics?id&start_date&end_date` | Window-scoped metrics |
| `Init Leads Pagination` → `Get all leads` loop | POST `/leads/list` | Paginated lead count + reply flag; capped at 5,000 |
| `Get all accounts` | GET `/accounts` | All sending accounts with warmup scores |
| `Get Lifetime Campaign Analytics` | GET `/campaigns/analytics` (creation→today) | Lifetime totals for CSV column |
| `Get Campaign Replies` | GET `/emails?email_type=received&mode=emode_focused` | Last 100 received emails |

**Lead pagination loop:** `Init Leads Pagination → Get all leads → Accumulate Leads → Check Loop End → [loop back or exit]`. The accumulator carries only counts and replier summaries — never full payloads — to stay memory-safe.

### Merge & Report Assembly

```
Merge (7 inputs) → Merge Report Data → Instantly Outbound Reporting AI Agent → Parse Instantly Output
```

**`Merge`** — `chooseBranch` mode with 7 inputs (added `Normalize Messaging KB` as input 6 in v1.2). Waits for all branches to complete before passing data downstream — a synchronization barrier; `Merge Report Data` reads each source by name, not from this node's own passthrough output.

**`Merge Report Data`** — Code node. The core aggregation step:
1. Indexes analytics, campaign data, leads, accounts, replies, and lifetime stats by campaign ID
2. Builds per-campaign `reportData` objects
3. **Filters `campaign_change_logs` to the current campaign** using `filterLogsByCampaign()` (fuzzy name match; logs without `campaign_name` are treated as client-wide) — including the `Campaign Context` lookup
4. **Attaches `priorMessagingContext`** — this client's recent `campaign_messaging_kb` entries across all campaigns/platforms, unfiltered by campaign (v1.2)
5. Filters replies to the reporting window, strips OOO/auto-replies
6. Looks up previous snapshot for trend context
7. Outputs one item per active campaign

**`Instantly Outbound Reporting AI Agent`** — Claude Sonnet 4.6 with adaptive thinking. Receives `reportData` JSON and outputs a structured JSON analysis object. Its system prompt has a "PRIOR MESSAGING CONTEXT" section (v1.2) instructing it to avoid re-recommending approaches `priorMessagingContext` shows already failed for this client, and to call out cross-campaign patterns when relevant. See README for verdict rules and tone guidelines.

**`Parse Instantly Output`** — Code node. Extracts the JSON from the AI text output, enforces a strict schema (all fields defaulted, no undefined), and attaches `platform: 'instantly'`.

---

## HeyReach (LinkedIn) Track

### Campaign Discovery

```
Check HeyReach Key → Retrieve all campaigns → Match Heyreach Campaign Name
                  ↘ [no key] Mock HeyReach Output + Mock HeyReach Lifetime
```

**`Retrieve all campaigns`** — POST `/campaign/GetAll` (offset 0, limit 100).

**`Match Heyreach Campaign Name - Exist or not`** — Filters to campaigns with status `IN_PROGRESS`, `active`, `1`, or `isActive: true`.

**`Wait - HeyReach API`** — 1-second pause before fanning out.

### Data Collection (parallel after Wait)

| Node | API Call |
|------|----------|
| `Retrieve found campaign data` | GET `/campaign/GetById?campaignId` |
| `Retrieve overall statistics for found campaign` | POST `/stats/GetOverallStats` (window-scoped) |
| `Retrieve all LinkedIn accounts` | POST `/li_account/GetAll` |
| `Retrieve inbox messages for found campaign` | POST `/inbox/GetConversationsV2` (campaignId filter, limit 100) |
| `Get HeyReach Lifetime Analytics` | POST `/stats/GetOverallStats` (creation→today) |

### Merge & Report Assembly

```
Merge all heyreach nodes data (6 inputs) → Merge Heyreach Nodes Data → Heyreach Outbound Reporting AI Agent → Parse HeyReach Output
```

**`Merge Heyreach Nodes Data`** — Code node equivalent to `Merge Report Data` but for LinkedIn:
1. Indexes campaign data and stats by campaign ID using `pairedItem` to correlate parallel branches
2. Classifies reply sentiment (positive / neutral / negative / informational) using keyword matching
3. Filters inbox conversations to replies that fall within the reporting window
4. **Filters `campaign_change_logs` to the current campaign** (same `filterLogsByCampaign()` helper as Instantly track)
5. **Attaches `priorMessagingContext`** — same client-wide KB lookup as the Instantly track (v1.2)
6. Outputs one item per active campaign

**`Heyreach Outbound Reporting AI Agent`** — Claude Sonnet 4.6. Same pattern as Instantly agent but with LinkedIn-specific schema, verdict rules, and the same "PRIOR MESSAGING CONTEXT" prompt section (v1.2).

**`Parse HeyReach Output`** — Enforces strict schema; attaches `platform: 'heyreach'`.

---

## Report Composition

```
Merge → Instantly + HeyReach (4 inputs)
  ├─ Parse Instantly Output        → input 0
  ├─ Parse HeyReach Output         → input 1
  ├─ Set → Instantly Lifetime Data → input 2
  └─ Set → HeyReach Lifetime Data  → input 3

→ Compose Combined Report → Has Active Campaigns? → Combined Readout AI Agent → Finalize Document
```

**`Compose Combined Report`** — Code node. Builds the full Slack message text and the CSV:
- Groups campaigns by clean name (strips client prefix, platform suffix, bracket tags)
- Renders reply highlights and key insights per campaign per platform
- Inserts `{{COMBINED_READOUT}}` placeholder for the AI synthesis step
- Generates the CSV with window + lifetime columns side by side: `"42 (187)"`
- Base64-encodes the CSV into a binary field for Slack upload

**`Has Active Campaigns?`** — Skips the AI readout and goes straight to `Loop End Check` if `campaignSummaries.length === 0`.

**`Combined Readout AI Agent`** — Claude Sonnet 4.6 (no adaptive thinking; speed-optimised). Writes exactly 3 sentences: cross-platform verdict, standout campaign, data-grounded conclusion. Hard rules: no numbers, no maturity caveats unless verdict === 'Early Stage', no recommendations.

**`Finalize Document`** — Replaces `{{COMBINED_READOUT}}` placeholder with the AI paragraph. Re-attaches the CSV binary from `Compose Combined Report`.

---

## Slack Delivery

```
Finalize Document → Request Slack Upload URL → Attach CSV Binary → Upload Binary to Slack → Complete Slack Upload & Share → Send Report Message to Slack
```

Uses Slack's two-step external upload API:
1. `files.getUploadURLExternal` — gets a pre-signed URL and `file_id`
2. POST binary to pre-signed URL
3. `files.completeUploadExternal` — shares the file to the channel with an initial comment
4. `chat.postMessage` (via n8n Slack node) — posts the main report text

---

## Snapshot & Knowledge Base Writes (parallel with Slack delivery)

```
Parse Instantly Output → Prepare Instantly Snapshot  → Write Instantly Snapshot to Supabase
Parse Instantly Output → Prepare Instantly KB Entry   → Write Instantly KB Entry to Supabase
Parse HeyReach Output  → Prepare HeyReach Snapshot    → Write HeyReach Snapshot to Supabase
Parse HeyReach Output  → Prepare HeyReach KB Entry     → Write HeyReach KB Entry to Supabase
```

Snapshots are upserted on `(campaign_id, reporting_window_start, reporting_window_end)` so re-runs are idempotent.

**`Prepare Instantly KB Entry` / `Prepare HeyReach KB Entry`** (v1.2) — Code nodes that build a `campaign_messaging_kb` row directly from that week's parsed AI output: `approach` is the primary signal plus what's-working/areas-to-watch, `outcome` is the executive summary, `verdict` is the AI verdict. No extra AI call — this reuses output already computed for the report and the snapshot write. `icp_segment` is left null (no ICP tagging exists upstream yet).

**`Write Instantly/HeyReach KB Entry to Supabase`** — POST to `campaign_messaging_kb` upserted on `(campaign_id, week_of)`, same idempotency pattern as the snapshot writes.

---

## Loop Control

```
Send Report Message to Slack → Loop End Check → [done or next client]
```

**`Loop End Check`** — Checks `$node['Loop active clients'].context['noItemsLeft']`. If false, loops back to `Loop active clients` to process the next client row.

---

## Friday Reminder Branch

Completely separate from the main flow. Fires at `0 16 * * 5`:

```
Schedule Trigger (Fri 4PM) → Fetch Active Clients (Reminder) → Build Friday Reminder Summary → Send Friday Reminder DM
```

Sends one DM listing all active clients and prompting analysts to log context before Monday.
