---
name: clickflow-decay-loop
description: Run ClickFlow's content-decay refresh loop from your own AI agent. Weekly, it reads back due refreshes (7/14/30-day GSC results), finds decaying pages, refreshes the highest-leverage ones into review-ready drafts, routes them to a human with clickable links, and records what worked so it compounds. Use when the user wants to install, run, or extend an automated "find + fix + measure decaying pages" loop over ClickFlow via MCP. All state lives in ClickFlow; chat memory is never load-bearing.
---

# ClickFlow Content-Decay Refresh Loop

Turn ClickFlow's decay detection + refresh engine into a standing, human-reviewed loop that runs inside **your** agent (Grok Build, Claude Code, Codex, a scheduled routine). ClickFlow does the heavy lifting (detect, refresh, measure) and remembers everything; you supply orchestration, judgment, and the human hand-off.

This skill is the operating manual. The identical instructions also ship as the MCP prompt **`content_decay_loop`** — if your client lists MCP prompts, invoking that prompt starts a cycle directly.

## Prerequisites

- ClickFlow MCP connected (`https://api.clickflow.com/v1/mcp`, OAuth or a `cf_ak_` API key) and an organization selected (`platform_list_organizations` → `platform_switch_organization`). If the `platform_*` tools aren't available in this session, STOP and tell the user to connect ClickFlow first — never improvise or invent results.
- A **Google Search Console** property connected in ClickFlow (decay detection and every readback are GSC-based). If not connected, tell the user to connect it first — the loop can't measure without it.

## Setup interview — ask BEFORE the first cycle (interactive runs only)

If a human is present and the loop isn't already configured (no prior cycle in this conversation, and `platform_get_loop_status()` shows zero tracked outcomes), ask these before touching anything — never guess them:

1. **Which organization?** Call `platform_list_organizations`, show the candidates, and have the user confirm — then `platform_switch_organization` and echo back the selected org name. Running the loop against the wrong org is the worst failure mode this skill has.
2. **How many refreshes per cycle?** Default 10; suggest 3–5 for the first cycle. Mention that each refresh consumes a content-plan slot.
3. **Drafts only, or apply to CMS after approval?** Default drafts-only. Only ask for a `cms_connection_id` if they want post-approval publishing.
4. **Readout + cadence:** where to deliver the weekly summary (this chat, Slack, email — whatever's connected), and whether to install the weekly schedule now.

Bake the answers into the scheduled prompt verbatim (org name + id, cap, delivery mode, readout channel) so future runs are self-contained. **On unattended/scheduled runs, never ask questions**: use the values baked into the prompt, or if missing, proceed with the defaults (current org, cap 10, drafts-only) and state the assumptions at the top of the readout.

## Step 0 — Check for an in-product routine (stand down on writes)

Before doing anything else, call `platform_get_loop_status()` and inspect its `in_product_routine` block. If it is present with `enabled: true`, this org's Content Refresh loop is already run by ClickFlow's **in-product routine**. **Stand down on every write step**: do not approve or dismiss suggestions, start refreshes, create roadmap items, or record outcomes/learnings — the routine owns those, and double-acting wastes content-plan slots on pages the routine is already handling. You may still run read-only steps (status, learnings, outcomes, a readout summary). Tell the user: "This org has the in-product Content Refresh routine enabled (next run: `next_run_at`) — manage it at https://app.clickflow.com/dashboard/automations. Running this external loop too would double-act on the same queue." Only proceed with writes if the user explicitly overrides (e.g. they are retiring the routine). Independently of the routine's status, skip any suggestion whose `claimed_by_routine` flag is `true` — a routine cycle has already claimed it.

## The core idea: ClickFlow is the memory, not the chat

A chat window doesn't survive 30 days and can't wake itself up. So **every run is re-entrant**: it starts by asking ClickFlow "what refreshes are now due for readback?" and measures them. A due readback waits safely in ClickFlow until *some* run — this immortal loop, a scheduled routine, or a brand-new chat weeks later — picks it up. Never store loop state in the conversation; always read it from the `platform_*` tools.

## Human gates — never cross without explicit approval

- **Never publish a content refresh without the user's go-ahead.** Default content refreshes to `delivery_mode='draft'`. The human reviews the draft, then approves publishing.
- **Internal-link fixes assume the CMS is connected, with a draft fallback.** Call `platform_list_cms_connections()`; if a connection exists, approve internal-linking suggestions with `delivery_mode='cms'` + that `cms_connection_id` — the workflow still pauses for keyword and link reviews, and the links approved there auto-apply (the link review IS the approval gate: curate it ruthlessly, rejecting forced anchors and off-topic source pages). No connection → fall back to `delivery_mode='draft'`, which returns manual link recommendations to hand the user.
- Cap refreshes per cycle (default **10**) so review stays manageable. Start at 3–5 on the first cycle to calibrate review capacity and content-plan limits, then scale — 10–20/week is a comfortable steady state for a mid-size site.
- **Never execute dismissals, deletions, redirects, noindex, or page consolidations.** You may (and should) *recommend* them in the readout when the evidence supports it — executing them is always the human's job, outside this loop.

## The weekly cycle (run in this order)

1. **Read back due refreshes FIRST** — this is the point of the loop.
   - `platform_list_refresh_outcomes(due_only=true)`.
   - For each, `platform_measure_refresh_outcome(outcome_id)` → returns a verdict (improved / flat / regressed) by diffing current GSC against the stored baseline.
   - For decisive results, `platform_record_loop_learning(...)` with a one-sentence lesson + the `refresh_outcome_id`.
2. **Load context** — `platform_get_loop_learnings()` (apply prior lessons) and `platform_get_loop_status()` (current picture).
3. **Find decay** — `platform_list_suggestions()` (persisted weekly queue). Optionally `platform_detect_content_decay()` for a fresh look, or `platform_get_page_performance(url)` to confirm a candidate.
4. **Triage — classify EVERY candidate before acting** (see `references/scoring.md` for the classes and evidence bars). Assign one of:
   - `protect` — winning or mid-readback page: do NOT touch it; name it in the readout so nobody else touches it either.
   - `refresh` — the loop executes this (as a draft).
   - `internal-link` — the loop executes this (approve the linking suggestion).
   - `net-new` — the loop may execute this as a DRAFT (`platform_create_roadmap_item(..., start_drafting=true)`) ONLY when all three prerequisites hold: no existing URL maps to the intent, a template/format is clear from what ranks, and an internal-link path exists. Counts against the per-cycle cap.
   - `consolidate` / `redirect` / `delete-noindex` — **recommendation-only, never executed.** Recommend only with evidence (e.g. two org pages splitting impressions on the same queries for consolidate; a page with ~zero impressions AND zero inbound value AND off-strategy for delete). Put these in the readout with the evidence, for the human to act on outside the loop.
   Then run the **pre-flight check** on everything classed `refresh`/`net-new` before burning a content slot: is it an article-type page (not a hub/service-index/utility page)? does it have a refresh thesis? does it cannibalize a sibling you're also acting on this cycle (if so, act on ONE and recommend consolidation for the pair)? was it already refreshed recently — check `platform_list_workflows` for a completed workflow on the same target URL (stale 'pending' suggestions re-offer already-drafted pages)? and is its `targetKeyword` sane — malformed keywords (numbering/query-syntax junk like "36.abm platform experts") become the draft's title, so drop or repair them before approving. Finally check quota: `platform_get_loop_status` reports `content_slots` — shrink the cycle to the remaining slots instead of failing mid-batch. Pick the top N survivors.
5. **Refresh** — content decay: `platform_approve_suggestion(suggestion_id, start_fix=true, delivery_mode='draft')` (from a suggestion) or `platform_optimize_page(content_url, delivery_mode='draft')` (from a URL). Internal linking: `delivery_mode='cms'` + the org's `cms_connection_id` when one exists (see Human gates), else `'draft'`. Poll `workflows_get_status`.
6. **Route for review** — every refresh returns a `review_url`; that server-minted link is the ONLY reviewable link, so never construct one yourself (guessed `/edit/...` URLs open empty documents). Collect it in this order: (a) the `platform_approve_suggestion` / `platform_optimize_page` response (board link while the workflow runs), (b) the completed workflow's output from `workflows_get_status` (`output.review_url` — the durable editor link, minted server-side on completion), (c) `platform_get_roadmap_item(roadmap_item_id).review_url`. Post them as clickable links: "N drafts ready: <links> — review & approve." Do not apply them yourself. If a completed draft has no `review_url` at all, say so explicitly and hand the user the `workflow_id` — do not improvise a URL. Before posting, spot-check each draft against `references/refresh-rubric.md` (or run the `refresh_review` prompt) and flag next to its link anything the human should not have to discover: product plugs or CTAs that aren't this org's, sentences repeated verbatim across this cycle's drafts, and statistics with no source. Flag, don't fix — the reviewer decides.
7. **Track for readback** — once the user confirms a refresh was applied, `platform_record_refresh_outcome(url, roadmap_item_id, workflow_id)` to store the baseline and schedule 7/14/30-day readbacks.
8. **Readout + reschedule** — summarize found / shipped / measured / **recommended** / next (or run the `weekly_readout` prompt). The recommended section carries the triage outputs the loop doesn't execute: protected pages (and why), consolidate/redirect/delete-noindex recommendations with their evidence, and anything skipped by pre-flight. If your environment can self-schedule, install a **weekly** cadence; otherwise ask the user to re-run next week.

## Self-scheduling

- **Any client with scheduling:** install a weekly cron/scheduled task pointing at the kickoff prompt (`kickoff.md`). That makes the loop effectively permanent — it survives closed windows because the state lives in ClickFlow, not the conversation. In Claude Code that's `/loop` or a scheduled routine; elsewhere use the equivalent.
- **Auth for unattended runs:** use a `cf_ak_` API key — interactive OAuth doesn't survive headless/cloud schedulers. claude.ai scheduled routines additionally need ClickFlow attached to the routine as a connector, or the run wakes up with no tools.
- **No scheduler available:** run manually each week. The readback still can't be missed — it's queued in ClickFlow.

## Bring your own tools

ClickFlow is the system of record, but use whatever else the user has connected for extra signal or delivery: their own GSC/Ahrefs MCP for corroborating trends, Slack to post the weekly readout, etc. Don't duplicate ClickFlow state elsewhere.

## Tool map

| Step | Tools |
|---|---|
| Readback | `platform_list_refresh_outcomes`, `platform_measure_refresh_outcome`, `platform_record_loop_learning` |
| Context | `platform_get_loop_learnings`, `platform_get_loop_status` |
| Find decay | `platform_list_suggestions`, `platform_detect_content_decay`, `platform_get_page_performance` |
| Refresh | `platform_approve_suggestion`, `platform_optimize_page`, `workflows_get_status` |
| Review | `platform_get_roadmap_item` (returns `review_url`), `platform_publish_to_cms` (only after approval) |
| Track | `platform_record_refresh_outcome` |

## References

- `references/scoring.md` — how to prioritize decaying pages.
- `references/refresh-rubric.md` — what a shippable refresh must contain (Google high-weight changes, AEO, humanization).
- `references/measurement.md` — the 7/14/30 readback + verdict rules.
- `kickoff.md` — the exact prompt to paste (or schedule) to start a cycle.
