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

**Status:** 🔲 Planned

### Problem
A reply in Instantly does not withdraw the lead from the HeyReach sequence, and vice versa. The same gap applies to tele-prospecting. A lead who has already replied on one channel continues to be messaged on the others, leading to:
- Duplicate or contradictory outreach to hot prospects
- Poor prospect experience
- Wasted sender reputation on leads already engaged

### Proposed solution
After collecting replies from each platform, match repliers across platforms by email address (Instantly → HeyReach) or LinkedIn profile URL (HeyReach → Instantly) and call the withdrawal API:

- **Instantly reply → stop in HeyReach:** Call `POST /campaign/StopLeadInCampaign` for each matched lead in active HeyReach campaigns
- **HeyReach reply → unsubscribe in Instantly:** Call `POST /leads/unsubscribe` or mark lead status in Instantly for each matched email

**Implementation notes:**
- Cross-match must be fuzzy (email normalisation: lowercase, trim)
- Should only withdraw from campaigns belonging to the same client
- Withdrawal should be logged to `campaign_change_logs` automatically with `change_category: 'Cross-Platform Withdrawal'`
- Consider a dry-run mode for initial rollout
- This gap calls live withdrawal APIs against active leads (real side effects), unlike Gaps 1 and 3 which are read/write against internal Supabase tables — treat as higher-risk and test in dry-run before enabling for real

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
