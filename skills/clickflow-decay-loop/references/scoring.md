# Scoring — triage every candidate, then pick what to refresh first

You get more candidates than your per-cycle cap. Classify first, then prioritize by expected recovered value, not by raw decline.

## Triage classes (assign one per candidate)

The loop only EXECUTES `refresh`, `internal-link`, and (with prerequisites) `net-new`. Refreshes and net-new are always drafts; internal-link fixes follow the CMS policy in SKILL.md (auto-apply via the connected CMS after the link review, draft fallback when no CMS). Everything else is a recommendation for the human, delivered in the readout with its evidence. Never dismiss suggestions; never execute consolidations, redirects, deletions, or noindex.

- **`protect`** — the page is growing, stable, or mid-readback from a recent refresh. Do nothing, and say so: naming protected pages stops anyone (including next week's run) from "fixing" a winner. A page whose clicks recovered after a refresh belongs here until its 30-day verdict.
- **`refresh`** — decaying, fixable, article-type page with a clear refresh thesis. The loop's bread and butter.
- **`internal-link`** — underlinked page with a strong source page available (the `internal_linking_opportunity` suggestions). Cheap, low-risk, execute freely within the cap.
- **`net-new`** — the queries deserve a page that doesn't exist. Prerequisites before drafting one: no existing URL cleanly maps to the intent (check `platform_list_published_content` / the site), the winning format is clear from what ranks, and there's an obvious internal-link path to it. Without all three, recommend instead of drafting.
- **`consolidate`** (recommend-only) — two or more org pages split impressions/clicks on substantially the same queries (e.g. sibling service pages both decaying on one keyword family). Evidence: overlapping query sets in GSC data. Recommend which page should win and which should merge into it.
- **`redirect`** (recommend-only) — the page's intent is fully served by another URL and it has meaningful inbound links/traffic worth preserving. Weaker evidence → leave alone.
- **`delete-noindex`** (recommend-only) — near-zero impressions AND no inbound value AND off-strategy. The bar is deliberately high; when in doubt, `protect` and move on.

## Signals (from `platform_list_suggestions` / `platform_get_page_performance`)

- **Traffic at stake** — a page that dropped from 2,000 → 1,500 clicks/mo matters far more than one that dropped 20 → 12. Weight by absolute clicks/impressions lost, not just percentage.
- **Severity + direction** — `metadata.clicksChange` and `ctrChange` (both negative for decay). Steeper + still-declining beats a page that already bottomed out.
- **Fixability** — can a high-weight refresh plausibly recover it? (See `refresh-rubric.md`.) A page that's thin, off-format, or missing sections the SERP rewards is highly fixable. A page decaying because the whole topic died, or because a competitor owns an unbeatable asset, is not.
- **Strategic value** — commercial/BOFU and service-adjacent pages over low-intent trivia. Reuse `platform_get_loop_learnings()` — if past cycles show a page type never recovers, deprioritize it.
- **Freshness of signal** — prefer suggestions detected recently; skip URLs you refreshed in the last ~45 days (they're mid-readback — check `platform_list_refresh_outcomes`).

## Rule of thumb

Score ≈ (clicks_at_stake) × (severity) × (fixability). Take the top N (default 10; use 3–5 on a first cycle). Skip anything where you can't articulate *what specific high-weight change* would move it — a refresh with no thesis wastes a review slot. Also skip hub, service-index, and utility pages (case-study hubs, privacy pages, category indexes): article-style refreshes destroy their page purpose even when GSC flags them as decaying.

## Don't

- Don't refresh a page that's decaying because it was intentionally deprioritized or is being consolidated elsewhere.
- Don't batch more than the cap — review capacity, not generation capacity, is the bottleneck.
- Don't re-refresh a page still inside its 7/14/30 readback window; let the measurement finish first.
