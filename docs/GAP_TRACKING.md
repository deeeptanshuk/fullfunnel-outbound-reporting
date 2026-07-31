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

Each campaign now receives only its own change logs. Logs without `campaign_name` are treated as client-wide and included for all campaigns.

### Prerequisite
`campaign_change_logs` table must have a `campaign_name` column (text, nullable). See [SUPABASE_SCHEMA.md](SUPABASE_SCHEMA.md).

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

---

## Gap 3 — No messaging knowledge base

**Status:** 🔲 Planned

### Problem
No repository of past messaging approaches, subject lines, or sequence structures by client, campaign, or ICP segment. Every new campaign starts from zero. Known successes and failures are lost between reporting cycles.

### Proposed solution
After each report run, write a structured entry to a `campaign_messaging_kb` Supabase table capturing:
- What messaging approach was in use (subject line themes, CTA type, sequence length)
- The AI verdict for that week
- The reply sentiment breakdown
- Key insights (What's Working / Areas to Watch)

At the start of a new campaign's first report, query the knowledge base for entries matching the same client or similar ICP segment and inject them into the AI agent prompt as prior context.

**Implementation notes:**
- Write step: add a node after `Finalize Document` that calls an AI agent to summarise the week's messaging approach and writes to `campaign_messaging_kb`
- Read step: add a node in the data collection phase that queries the KB and includes it in `reportData` for the AI agents
- Schema: see `campaign_messaging_kb` placeholder in [SUPABASE_SCHEMA.md](SUPABASE_SCHEMA.md)
