# Gap Tracking

Identified gaps in the outbound reporting system, their status, and implementation notes.

Source: *Outbound Change Tracking — Problem & Gaps* (Deep / Hunter Brayton, July 30 2026)

---

## Gap 1 — Change logs not broken out by campaign

**Status:** ✅ Resolved in v1.1 (2026-07-31)

### Problem
Messaging changes, prompts, and analyst logs were organised by client only. When a client had multiple active campaigns, every campaign's AI agent saw the full client-wide change log. This caused:
- Recommendations for Campaign B being incorrectly suppressed because of a change made to Campaign A
- The `Campaign Context` note from one campaign appearing in another campaign's report

### Root cause
`Fetch Campaign Change Logs` fetches all logs for a client. In `Merge Report Data` and `Merge Heyreach Nodes Data`, `changeLogs` was passed wholesale to each campaign's `reportData` without any per-campaign filtering.

### Fix
Added `filterLogsByCampaign(logs, campaignName)` helper in both `Merge Report Data` and `Merge Heyreach Nodes Data`:

```js
function filterLogsByCampaign(logs, campaignName) {
  if (!campaignName || !logs.length) return logs;
  const norm = n => (n || '').toLowerCase().replace(/[^a-z0-9]/g, '');
  const cn = norm(campaignName);
  return logs.filter(log => {
    if (!log.campaign_name) return true; // client-wide logs pass through
    const ln = norm(log.campaign_name);
    return ln === cn || cn.includes(ln) || ln.includes(cn);
  });
}
```

Each campaign now receives only its own change logs — including the `Campaign Context` lookup, which previously scanned the unfiltered list and could surface another campaign's note. Logs without `campaign_name` are treated as client-wide and included for all campaigns.

### Prerequisite
`campaign_change_logs` table must have a `campaign_name` column (text, nullable). See [SUPABASE_SCHEMA.md](SUPABASE_SCHEMA.md).

### Note — live/GitHub drift (found & fixed 2026-07-31, v1.2)
The live n8n workflow was built before this fix landed and had never received it — `filterLogsByCampaign` existed in the GitHub JSON but not in the actual running workflow. This was discovered and corrected while implementing Gap 3 (same two code nodes needed touching either way). Live and GitHub are now in sync.

---

## Gap 2 — No unified cross-platform lead withdrawal

**Status:** 🟡 Dry-run implemented in v1.3 (2026-08-01) — not yet calling live withdrawal APIs

### Problem
A reply in Instantly does not withdraw the lead from the HeyReach sequence, and vice versa. The same gap applies to tele-prospecting. A lead who has already replied on one channel continues to be messaged on the others, leading to:
- Duplicate or contradictory outreach to hot prospects
- Poor prospect experience
- Wasted sender reputation on leads already engaged

### Why this shipped as dry-run only, and as a fuzzy match
The original proposed solution below assumed email (Instantly) and LinkedIn URL (HeyReach) could be cross-referenced directly. In practice, neither platform's data as currently collected by this workflow carries the other's identifier — Instantly's reply/lead data has no LinkedIn URL field, and HeyReach's reply/lead data has no email field. There is no shared identifier to match on with high confidence today.

Given that, and given this gap's real side effects (stopping a live sequence, unsubscribing a real lead) compared to Gaps 1 and 3's internal-table-only reads/writes, this pass implements **detection and logging only** — no withdrawal API is called. Matching is done by normalised name + company (fuzzy, medium confidence), and every match is logged to `campaign_change_logs` and posted to Slack labelled `DRY RUN`, so the team can verify accuracy against real weeks of data before any live withdrawal is built.

**Tele-prospecting is out of scope for this pass** — no tele-prospecting tool/API was identified as in use, so the withdrawal loop currently only covers Instantly ↔ HeyReach.

### What was built (v1.3)
**Data collection extended (needed for the roster side of the match):**
- `Accumulate Leads` (Instantly) now also accumulates a lightweight full lead roster — `email`/`name`/`company` only, for every lead, not just repliers — capped alongside the existing 5,000-lead limit. This stays memory-safe; it's 3 short strings per lead, not a full payload.
- `Merge Heyreach Nodes Data` now exposes `leadsAnalysis.connectedLeads` (previously only its count was surfaced) — everyone with an active HeyReach conversation thread, used as the HeyReach-side roster.

**New node: `Cross-Platform Withdrawal Check`** — runs once per client (after `Merge – Instantly + HeyReach`, in parallel with `Compose Combined Report`). For every Instantly replier this window, checks if their normalised name+company appears in the client's HeyReach roster (`connectedLeads`) — if so, that's a dry-run "would stop in HeyReach" candidate. Mirrors the check in the other direction (HeyReach replier → Instantly roster → "would unsubscribe from Instantly"). Emits zero items when there's nothing to flag (no separate empty-case handling needed — downstream nodes simply don't run).

**New nodes: `Log Dry-Run Withdrawal to Supabase`** (POST to `campaign_change_logs`, `change_category: 'Cross-Platform Withdrawal (Dry Run)'`, `actioned_by: 'Automated (Dry Run)'`) and **`Notify Dry-Run Withdrawal Match`** (Slack message to the client's channel, clearly labelled DRY RUN, using the same native Slack node + channel-ID handling as `Send Report Message to Slack`).

### Known limitation — this only catches "still enrolled," not literally "every possible lead"
The HeyReach roster (`connectedLeads`) comes from inbox conversation data (`GetConversationsV2`), which covers everyone with an active conversation thread — a good proxy for "currently being messaged," not a literal export of every lead in every campaign. The Instantly roster is the full paginated lead list (up to 5,000), which is closer to complete. Confidence is always reported as "medium (name+company fuzzy match)" — there is no "high confidence / exact identifier" tier available with the data collected today.

### Original proposed solution (superseded by the above where it assumed a shared identifier)
- **Instantly reply → stop in HeyReach:** Call `POST /campaign/StopLeadInCampaign` for each matched lead in active HeyReach campaigns
- **HeyReach reply → unsubscribe in Instantly:** Call `POST /leads/unsubscribe` or mark lead status in Instantly for each matched email

**Next step, once dry-run has been verified against real data:** wire the two API calls above behind the existing match detection, replacing the log/notify-only branch (or adding to it) with the actual withdrawal call — gated so it only fires above some confidence bar the team is comfortable with.

---

## Gap 3 — No messaging knowledge base

**Status:** ✅ Resolved in v1.2 (2026-07-31)

### Problem
No repository of past messaging approaches, subject lines, or sequence structures by client, campaign, or ICP segment. Every new campaign starts from zero. Known successes and failures are lost between reporting cycles.

### Fix
Added a `campaign_messaging_kb` Supabase table (see [SUPABASE_SCHEMA.md](SUPABASE_SCHEMA.md)) with a read step and a write step in the live workflow:

**Write step** — `Prepare Instantly KB Entry` / `Prepare HeyReach KB Entry` run off `Parse Instantly Output` / `Parse HeyReach Output` (parallel to the existing snapshot-write branch) and derive `approach` (primary signal + what's working / areas to watch), `outcome` (executive summary), and `verdict` directly from that week's AI agent output — no extra AI call. `Write Instantly/HeyReach KB Entry to Supabase` upsert on `(campaign_id, week_of)`.

**Read step** — `Fetch Messaging KB` queries the client's most recent 15 KB entries across **all** of their campaigns and platforms (deliberately not filtered to the current campaign, unlike change logs — the point is cross-campaign learning). `Normalize Messaging KB` parses the response the same way `Normalize Client History` does. Both feed into `Merge` / `Merge all heyreach nodes data` as a new synchronization input, and `Merge Report Data` / `Merge Heyreach Nodes Data` attach the result to `reportData.priorMessagingContext`.

Both AI agent system prompts (`Instantly Outbound Reporting AI Agent`, `Heyreach Outbound Reporting AI Agent`) now have a "PRIOR MESSAGING CONTEXT" instruction block telling the model to avoid re-recommending approaches already shown to fail for this client, and to call out cross-campaign/cross-platform patterns when relevant.

**Not included in this pass:**
- `icp_segment` is not populated — there's no ICP tagging anywhere upstream yet (`client_configs`, `campaign_change_logs`). The column exists in the schema for when that's added.
- No dedicated summarisation AI call — the KB entry is derived from the existing structured AI output fields, which keeps this cheap and avoids adding a model call that wasn't clearly needed yet. If `approach`/`outcome` prove too thin in practice (e.g. teams want actual subject-line text captured), revisit with a dedicated summarisation step.
