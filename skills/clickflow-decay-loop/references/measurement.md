# Measurement — the 7/14/30-day readback

This is the loop's whole reason to exist: prove (or disprove) that a refresh recovered traffic, and feed that back so the loop learns. All of it is durable in ClickFlow — it does not depend on any chat staying open.

## How it works

1. **Baseline** — when a refresh is applied, call `platform_record_refresh_outcome(url, roadmap_item_id, workflow_id)`. ClickFlow captures the pre-refresh GSC totals (last 28 days) as `before` and schedules readbacks at **7, 14, and 30 days**.
2. **Readback** — on any later run, `platform_list_refresh_outcomes(due_only=true)` returns outcomes whose next readback time has passed. For each, `platform_measure_refresh_outcome(outcome_id)`:
   - re-queries GSC for the same window length as the baseline,
   - computes deltas (clicks %, CTR points, position),
   - assigns a verdict and advances the schedule (7 → 14 → 30).

## Verdict rules (computed server-side; documented here for transparency)

Based on percent change in clicks vs the stored baseline:

- **improved** — clicks up **≥ +10%**. Decisive: it sticks, and no further readbacks run for that outcome.
- **regressed** — clicks down **≤ −10%**. Decisive: sticks; surface it — the refresh may have hurt, or an external factor is at play.
- **flat / measured** — in between. Keeps measuring at the next checkpoint; if still not decisive at the **30-day** (final) readback, it locks as **flat**.

Position is reported as an absolute delta (negative = better rank); CTR as percentage points. Use these to add color, but the click-based verdict is the headline.

## Turning results into learnings

After a decisive readback, record a one-sentence lesson with `platform_record_loop_learning(learning, refresh_outcome_id=...)`. Good learnings are specific and reusable next cycle:

- "Comparison pages recovered clicks within 14 days when we added a comparison table + updated the title; guide pages took the full 30."
- "Date-bump-only refreshes (no high-weight change) stayed flat — stop proposing them."
- "Refreshing pages under 300 clicks/mo rarely moved the needle; prioritize higher-traffic decay."

`platform_get_loop_learnings()` surfaces these at the start of the next cycle so the loop compounds instead of relearning.

## Honesty

- Readbacks need real elapsed time — a 7-day verdict can't exist until 7 days after apply. Don't fabricate results; report "not yet due."
- GSC lags ~3 days, so windows end 3 days before today. A brand-new refresh has no signal for at least a week — that's expected.
- Attribution isn't perfect (seasonality, algorithm updates, other changes). Report the delta honestly; a clear win or clear regression is signal, noise near zero is "flat."
