# Gap Tracking

Identified gaps in the outbound reporting system, their status, and implementation notes.

Source: *Outbound Change Tracking — Problem & Gaps* (Deep / Hunter Brayton, July 30 2026)

---

## Gap 1 — Change logs not broken out by campaign

**Status:** ✅ Resolved in v1.4 (2026-08-01) — see correction below; earlier "resolved" status in v1.1/v1.2 was against a schema that turned out to be wrong. Re-confirmed live in v1.5 (2026-08-04) against a second client.

### Problem
Messaging changes, prompts, and analyst logs were organised by client only. When a client had multiple active campaigns, every campaign's AI agent saw the full client-wide change log. This caused:
- Recommendations for Campaign B being incorrectly suppressed because of a change made to Campaign A
- The `Campaign Context` note from one campaign appearing in another campaign's report

### Root cause
`Fetch Campaign Change Logs` fetches all logs for a client. In `Merge Report Data` and `Merge Heyreach Nodes Data`, `changeLogs` was passed wholesale to each campaign's `reportData` without any per-campaign filtering.

### Fix history — and a schema correction (important)
v1.1/v1.2 added a `filterLogsByCampaign(logs, campaignName)` helper matching on a `campaign_name` field. **This was built against a documented schema that did not match the real table.** A live `information_schema.columns` query against `campaign_change_logs` (2026-08-01) showed:

```
id, client_name, campaign_id, platform, change_category, change_description, actioned_by, actioned_at
```

There is **no `campaign_name` column at all** — `campaign_id` and `platform` are the real join keys, and both are `NOT NULL`. Since every row's `campaign_name` field was `undefined` against the real data, the v1.1/v1.2 filter silently treated *every* log as client-wide — the bug this gap was meant to fix was never actually fixed in production, despite the code looking correct.

**v1.4 fix** — replaced with `filterLogsForCampaign(logs, campaignIdValue, platform)` in both `Merge Report Data` and `Merge Heyreach Nodes Data`:

```js
function filterLogsForCampaign(logs, campaignIdValue, platform) {
  if (!logs.length) return logs;
  return logs.filter(log => {
    if (log.platform === 'general') return true; // client-wide notes pass through
    return log.platform === platform && String(log.campaign_id) === String(campaignIdValue);
  });
}
```

`platform = 'general'` is the existing convention (already used by the Slack Command Handler's bulk-context flow) for notes that apply to the whole client rather than one campaign — this replaces the old "missing campaign_name" pass-through rule.

**Important asymmetry to know about:** Instantly's real numeric campaign ID does not survive the AI output schema upstream (`campaignOverview` never carried an `id` field), so `campaign_snapshots` — and therefore every Slack-logged Instantly change — has always stored the **campaign name** as `campaign_id` for Instantly rows. HeyReach rows use the real numeric HeyReach campaign ID. The v1.4 fix matches this existing convention rather than trying to introduce a "more correct" numeric ID that was never actually being captured anywhere in the pipeline. `Merge Report Data` matches Instantly logs against `targetCampaignName`; `Merge Heyreach Nodes Data` matches HeyReach logs against the real numeric `matchedCampaignId` **or** `targetCampaignName` (see "Related fixes" below for why both are accepted).

**v1.5 re-confirmation (2026-08-04):** re-ran this test live against a second client (CONSTELLATION), planting a campaign-specific note on its Instantly campaign and a client-wide note, and confirmed the campaign-specific note correctly stayed scoped to its own campaign with no leakage into a different active campaign's report. Also confirmed this name-vs-ID asymmetry is a deliberate, working design choice rather than a bug: it's inherent to using a human-readable name as an identifier (see "Known edge cases" below), and switching Instantly to a numeric ID would require migrating every existing name-keyed row and is not worth the churn given it isn't actually broken.

**Known edge cases from the name-based identifier (theoretical, not observed as an active bug):**
- If an Instantly campaign is renamed, its next report is treated as a new campaign under the new name — it won't find its own prior snapshot history or pre-rename change-log notes. No crash, no data corruption, just a soft reset in continuity.
- If a client ever had two active Instantly campaigns with the exact same name at the same time, they would collide: both would receive the same change-log notes, and if they reported the same week, one snapshot row could silently overwrite the other. Not seen in any client's real data so far.

### Prerequisite
None — `campaign_id` and `platform` already exist on the real table (confirmed live). No migration needed for Gap 1 itself.

---

## Gap 2 — No unified cross-platform lead withdrawal

**Status:** 🟡 Dry-run implemented in v1.4 (2026-08-01), upgraded from the v1.3 fuzzy-only approach — not yet calling live withdrawal APIs. Not yet tested end-to-end with a real reply as of v1.5 (2026-08-04).

### Problem
A reply in Instantly does not withdraw the lead from the HeyReach sequence, and vice versa. The same gap applies to tele-prospecting. A lead who has already replied on one channel continues to be messaged on the others, leading to:
- Duplicate or contradictory outreach to hot prospects
- Poor prospect experience
- Wasted sender reputation on leads already engaged

**Tele-prospecting is out of scope** — no tele-prospecting tool/API was identified as in use, so the withdrawal loop only covers Instantly ↔ HeyReach.

### v1.3 → v1.4: discovery of `campaign_leads`
v1.3 shipped with fuzzy name+company matching only, because the data each n8n workflow collects directly from the Instantly/HeyReach APIs has no shared identifier (Instantly reply data has email, no LinkedIn URL; HeyReach reply data has LinkedIn URL, no email).

While verifying schema assumptions live against Supabase, a **third table — `campaign_leads` — was discovered**, populated by some process independent of either n8n workflow (neither writes to it; `last_synced_at` shows it's actively and recently synced, likely via Clay or a dedicated sync job). Its real schema:

```
id, client_name, platform, campaign_id, campaign_name, external_lead_id,
email, first_name, last_name, company_name, company_domain,
linkedin_profile_url, lead_status, last_synced_at, created_at
```

Confirmed from live sample rows: Instantly rows have real `email` (no `linkedin_profile_url`); HeyReach rows have `linkedin_profile_url` and — in most sampled rows — `email` too. `lead_status` uses each platform's native vocabulary: HeyReach text (`Finished`/`InSequence`/`Pending`/`Failed`/`Excluded`), Instantly numeric codes (`1`/`3`/`-1`/`-2`/etc., roughly Active/Completed/Bounced/Unsubscribed).

**This is a real shared identifier this workflow didn't have access to before** — the original Gap 2 proposal's "match by email" assumption is achievable after all, just via this table rather than directly from live API pulls.

### What was built (v1.4)
**New nodes:** `Fetch Campaign Leads` (queries `campaign_leads` for the client), `Normalize Campaign Leads` (same parsing pattern as the other Supabase list-fetches) — wired into both merge barriers (`Merge` now 8 inputs, `Merge all heyreach nodes data` now 7).

**`Cross-Platform Withdrawal Check` rewritten** with a two-tier match:
1. **High confidence (email match):** an Instantly replier's email is looked up directly against HeyReach's `campaign_leads` rows for the client; if found and that lead's `lead_status` is non-terminal (still actively enrolled), it's flagged. Mirrored for HeyReach repliers: their `campaign_leads` (HeyReach) row is looked up by LinkedIn URL to recover their email, which is then checked against Instantly's `campaign_leads` rows.
2. **Medium confidence (name+company fuzzy match):** falls back to the v1.3 approach — comparing this run's own live roster data (`allLeadsLite`, `connectedLeads`) — only when no email match exists (e.g. a HeyReach lead with no email on file).

Every match carries a `confidence` label (`high (email match)` or `medium (name+company fuzzy match)`), visible in both the Slack notification and the `campaign_change_logs` entry, so matches can be trusted or scrutinized accordingly.

**`Log Dry-Run Withdrawal to Supabase`** — POSTs to `campaign_change_logs` using the real schema (`campaign_id`, `platform` — not `campaign_name`, which doesn't exist; see Gap 1's correction above).

**`Notify Dry-Run Withdrawal Match`** — posts to the client's Slack channel, clearly labelled DRY RUN. **Known open item:** intended to redirect to a dedicated test channel during the verification period rather than the client's own report channel — pending a channel ID.

No withdrawal API (`StopLeadInCampaign`, Instantly unsubscribe) is called in this pass.

### Data retained from v1.3 (used as the medium-confidence fallback)
- `Accumulate Leads` (Instantly) still accumulates a lightweight full lead roster (`email`/`name`/`company`, every lead not just repliers) alongside its existing counts/repliers, capped at 5,000
- `Merge Heyreach Nodes Data` still exposes `leadsAnalysis.connectedLeads` (everyone with an active HeyReach conversation thread)

### Known limitations
- The medium-confidence fallback still only approximates "currently enrolled" (HeyReach's `connectedLeads` covers active conversation threads, not a literal export of every lead in every campaign)
- `campaign_leads`' own completeness/freshness depends entirely on whatever external process populates it — not verified as part of this work
- Live withdrawal calls are still not wired up; this remains detection + logging only
- Still not exercised end-to-end with a real reply as of v1.5 — needs a real test lead added to a live Instantly campaign, a real reply, and a mirrored `campaign_leads` row on the HeyReach side

### Original proposed solution
- **Instantly reply → stop in HeyReach:** Call `POST /campaign/StopLeadInCampaign` for each matched lead in active HeyReach campaigns
- **HeyReach reply → unsubscribe in Instantly:** Call `POST /leads/unsubscribe` or mark lead status in Instantly for each matched email

**Next step, once dry-run has been verified against real data (especially the high-confidence email matches):** wire the two API calls above behind the existing match detection — likely gated to the `high (email match)` tier only at first, given the medium-confidence fuzzy tier is still a guess.

---

## Gap 3 — No messaging knowledge base

**Status:** ✅ Implemented in v1.2 (2026-07-31), table created and confirmed working live in v1.5 (2026-08-04) after two real bugs found during testing were fixed.

### Problem
No repository of past messaging approaches, subject lines, or sequence structures by client, campaign, or ICP segment. Every new campaign starts from zero. Known successes and failures are lost between reporting cycles.

### Fix
Added read/write steps for a `campaign_messaging_kb` Supabase table (see [SUPABASE_SCHEMA.md](SUPABASE_SCHEMA.md)) in the live workflow:

**Write step** — `Prepare Instantly KB Entry` / `Prepare HeyReach KB Entry` run off `Parse Instantly Output` / `Parse HeyReach Output` (parallel to the existing snapshot-write branch) and derive `approach` (primary signal + what's working / areas to watch), `outcome` (executive summary), and `verdict` directly from that week's AI agent output — no extra AI call. `Write Instantly/HeyReach KB Entry to Supabase` upsert on `(campaign_id, week_of)`.

**Read step** — `Fetch Messaging KB` queries the client's most recent 15 KB entries across **all** of their campaigns and platforms (deliberately not filtered to the current campaign, unlike change logs — the point is cross-campaign learning). `Normalize Messaging KB` parses the response the same way `Normalize Client History` does. Both feed into `Merge` / `Merge all heyreach nodes data` as a synchronization input, and `Merge Report Data` / `Merge Heyreach Nodes Data` attach the result to `reportData.priorMessagingContext`.

Both AI agent system prompts (`Instantly Outbound Reporting AI Agent`, `Heyreach Outbound Reporting AI Agent`) have a "PRIOR MESSAGING CONTEXT" instruction block telling the model to avoid re-recommending approaches already shown to fail for this client, and to call out cross-campaign/cross-platform patterns when relevant.

### Two real bugs found during v1.5 testing, both fixed
1. **Silent pipeline stall on an empty KB.** A client's very first report (no messaging history yet) caused `Fetch Messaging KB` to return zero rows, which made n8n skip `Normalize Messaging KB` entirely (standard n8n behavior for a node fed zero input items), starving one of the `Merge` node's required inputs and silently stopping the whole report before it ever reached Slack — with no error thrown anywhere. Fixed by setting `alwaysOutputData: true` on `Fetch Messaging KB`, so it always emits an item (including the existing `is_empty_kb` marker) even on a truly empty response.
2. **Multi-row parsing bug.** Even with #1 fixed, `Normalize Messaging KB`'s old code (`kbItems[0]?.json`, checking whether that was already an array or `.data`-wrapped) assumed the first fetched item's `json` already contained every row. In reality, Supabase's JSON array response gets split into one n8n item per row, so this only ever inspected the first row and silently fell back to `is_empty_kb: true` even when real KB history existed — confirmed via a live execution where Supabase returned exactly 1 real row, but `priorMessagingContext` still came out empty. Fixed to `kbItems.map(item => item.json).filter(Boolean)`, reading back every returned row.

The same multi-row parsing bug pattern was later found and fixed in `Normalize Client History` (see "Related fixes" below) — that one fed `previousSnapshot`, a different feature, not this KB.

### Not included in this pass
- `icp_segment` is not populated — there's no ICP tagging anywhere upstream yet (`client_configs`, `campaign_change_logs`). The column exists in the schema for when that's added.
- No dedicated summarisation AI call — the KB entry is derived from the existing structured AI output fields, which keeps this cheap and avoids adding a model call that wasn't clearly needed yet. If `approach`/`outcome` prove too thin in practice (e.g. teams want actual subject-line text captured), revisit with a dedicated summarisation step.

---

## Related fixes — not part of the original 3 gaps

Found and fixed during v1.5 (2026-08-04) live testing. None of these were requested in the original problem doc; they surfaced while verifying the gaps above actually work end to end.

### HeyReach `campaign_id` fallback bug (see Gap 1's v1.5 note and the CHANGELOG for detail)
`Prepare HeyReach Snapshot` was falling back to the campaign name instead of the real numeric ID, because the AI agent's structured output doesn't reliably carry that field through. Fixed by sourcing the ID from `Match Heyreach Campaign Name - Exist or not` instead. This bug predates Gap 1's fix and existed in production before this testing round; it directly threatened Gap 1's HeyReach-side guarantee (see Gap 1 above).

### HeyReach change logs could be permanently lost during the ID migration
The fix above created a new, real risk: the Slack Command Handler's campaign dropdown builds its value from whatever's currently in `campaign_snapshots.campaign_id`. A change logged via Slack *before* a given campaign's snapshot refreshed under the new numeric ID would be saved under the old campaign name instead — and since `campaign_change_logs` is write-once (unlike `campaign_snapshots`, refreshed every week), that note would never match the report filter again, silently, forever. Caught before it caused any real data loss. Fixed by having `Merge Heyreach Nodes Data`'s `filterLogsForCampaign` accept either the real numeric ID or the campaign's current name, so a note logged in either era is still picked up correctly. No manual data migration needed.

### `previousSnapshot` (week-over-week trend comparison) never worked, for anyone, on either platform
Two independent bugs: a multi-row parsing bug in `Normalize Client History` (same pattern as Gap 3's KB bug above) meant history was always read as empty; and, for Instantly specifically, a separate identifier mismatch in `Merge Report Data`'s lookup meant even after the parsing fix, Instantly's `previousSnapshot` still couldn't match against real Instantly history (HeyReach's lookup already used the correct identifier and needed no change). Both fixed — full detail in `CHANGELOG.md`'s v1.5 entry.

### Change-log duplication (up to 11x) reaching the AI
`Fetch Campaign Change Logs` sits downstream of `Fetch Client History` in the node graph, causing it to redundantly re-run once per upstream item and produce many duplicate copies of the same rows. Confirmed correctness of the per-campaign filtering itself was unaffected — this was pure duplicate content and wasted AI tokens. Fixed with a dedupe-by-row-id step in both merge nodes.
