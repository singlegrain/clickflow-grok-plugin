# Kickoff prompt

Paste this to start a cycle (or point your scheduler at it). It assumes the ClickFlow MCP is connected, an org is selected, and GSC is connected. If your client supports MCP prompts, you can instead just invoke the **`content_decay_loop`** prompt — same instructions.

---

Load the `clickflow-decay-loop` skill if it's installed (otherwise follow the server's `content_decay_loop` MCP prompt, or the steps below — same instructions) and run ONE cycle of the ClickFlow content-decay refresh loop. If this is our first cycle, start by confirming with me: which organization (list them — don't guess), how many refreshes this cycle (default 10, suggest 3–5 first time), drafts-only or CMS after approval, and where to send the readout — then bake my answers into the weekly schedule so future runs don't need to ask. In short:

1. First, read back any refreshes now due for their 7/14/30-day check: `platform_list_refresh_outcomes(due_only=true)` → `platform_measure_refresh_outcome` each → record a learning for decisive results.
2. Load context: `platform_get_loop_learnings()` and `platform_get_loop_status()`.
3. Find decaying pages: `platform_list_suggestions()` (persisted weekly queue).
4. Triage every candidate per the scoring rules — protect / refresh / internal-link / net-new / consolidate / redirect / delete-noindex — then pick the top 10 executable ones (drop to 3–5 if this is my first cycle). Pre-flight each pick: skip URLs already refreshed recently (check `platform_list_workflows` — stale 'pending' suggestions re-offer drafted pages) drop/repair malformed target keywords (they become the draft's title), and size the batch to the remaining `content_slots` from `platform_get_loop_status`. Only refresh, internal-link, and prerequisite-passing net-new get executed (as drafts); consolidate/redirect/delete are recommendations for me, with evidence.
5. Refresh content-decay picks as DRAFTS (`platform_approve_suggestion(..., delivery_mode='draft')` or `platform_optimize_page(..., delivery_mode='draft')`). For internal-linking picks, check `platform_list_cms_connections()`: if a connection exists, approve with `delivery_mode='cms'` + that `cms_connection_id` so links you approve at the link review auto-apply; otherwise fall back to draft (manual recommendations). Poll `workflows_get_status`.
6. Post me the `review_url` for each finished draft as clickable links (from the tool responses / completed workflow output — never construct URLs yourself), flagging next to each anything I shouldn't have to discover (foreign product plugs, boilerplate repeated across this cycle's drafts, unsourced stats) — do NOT publish anything. I'll review and approve.
7. When I confirm a refresh is applied, call `platform_record_refresh_outcome(...)` so the 7/14/30-day readback is scheduled.
8. Give me a short readout (found / shipped / measured / recommended / next — recommended = protected pages plus any consolidate/redirect/delete suggestions with their evidence), then install yourself on a weekly schedule if you can; otherwise remind me to re-run next week.

Hard rules: if the ClickFlow MCP tools (`platform_*`) aren't available in this session, STOP before step 1 and tell me to connect ClickFlow first (never improvise or invent results); never publish a content refresh without my explicit approval (internal-link fixes may auto-apply via the connected CMS after you curate the link review — that review is the approval gate); never dismiss suggestions; never execute consolidations, redirects, deletions, or noindex — recommend them only; at most 10 refreshes this cycle (3–5 on a first run); treat ClickFlow as the source of truth (not this chat). You may use my other connected tools (GSC, Ahrefs, Slack) for extra signal or to send the readout.

---

## One-paste install (what the dashboard card hands you)

The **Content Decay Autopilot** card in ClickFlow → API & MCP (the "Install a loop in your agent" section) gives you a per-harness copy-paste block that, run once in your agent, will: verify the ClickFlow MCP connection (handing you the exact connect command if it's missing), kick off the first cycle using the server's `content_decay_loop` prompt as the playbook, and schedule a weekly run. This downloadable skill is the optional local twin of that prompt — same instructions, plus the scoring and review-rubric references. After install it's hands-off — drafts show up for your review each week and results are measured automatically.
