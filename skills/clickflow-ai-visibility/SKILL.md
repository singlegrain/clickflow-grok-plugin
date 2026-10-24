---
name: clickflow-ai-visibility
description: Measure and improve how a brand shows up inside AI assistants — position versus competitors in ChatGPT and Perplexity, topic-by-topic visibility, sentiment themes in how assistants describe the brand, and the AI-referred traffic actually landing on the site (AEO). Use when the user asks about AI visibility, answer-engine optimization, whether ChatGPT recommends them, how they compare to competitors in AI answers, or where AI traffic is coming from.
---

# AI visibility and answer-engine optimization

Two different questions, two different data sources. Keep them apart — conflating them produces
confident nonsense.

| Question | Tools | Source |
|---|---|---|
| Do sampled responses mention us, and how do we rank against competitors? | `platform_get_visibility_*` | ClickFlow collector samples; ChatGPT via Oxylabs, current Perplexity collector via Sonar API |
| Is AI-referred traffic actually arriving on the site? | `platform_get_aeo_data` | Google Analytics 4 |

Read `clickflow-seo` first for org selection. Everything here is read-only, so orient freely.

## Check the scan first

`platform_get_visibility_status(history_limit=10)` returns the latest scan's state — `running`, `completed`, or
`failed` — with a timestamp. Call it before reporting numbers. Reporting a stale or partial scan as
current is the main way this goes wrong. If it's `running`, say so and give the timestamp of the
data you're describing; if `failed`, report that rather than presenting whatever's cached as fresh.

The latest attempted scan and the latest completed rankings can be different runs. Match the
status `_id` to rankings `runId` before applying completeness metadata to rankings. Always report
the rankings' own `completedAt`, even when a newer scan is running or failed. If they do not match,
the older rankings' completeness is unknown unless its exact run appears in the returned history.

Inspect `analysisFailures`, `scrapeCoverage`, `resultsExpected`, and `resultsStored` when present.
The storage counts describe prepared result documents and acknowledged writes, not all configured
prompts. A write timeout can leave unacknowledged rows: do not claim these are an exact missing-row
count. `scrapeCoverage` is the worst platform's coverage, not a per-platform breakdown. Missing
fields on older deployments or runs mean **unknown**, never zero failures or full coverage.
A completed run alone does not certify a complete sample.

If the server tool schema does not yet support `history_limit`, call the tool without it and
report history/provenance as unavailable. Do not invent a successful history response.

The optional `history_limit` (1–50; 0 means latest only) returns newest-first run records from
one organization-scoped snapshot, including failed/running attempts. Each record has its own ID,
status, dates, available counts and `measurementSources`. Never treat a failed attempt as a
completed comparison. Rankings' `previousRunId`/`previousCompletedAt` identify the actual baseline;
use these IDs, not adjacent entries in mixed-status history. No previous run means no measured change.

`measurementSources` groups acknowledged documents by platform/source/model. `resultCount` is the
group size; `observedCount` is how many have a response-received timestamp, with
`firstObservedAt`/`lastObservedAt` bounding those observations. These are collector receipt times,
not search-engine crawl dates or database write times. A missing source/model remains unknown.
A changed source/model, sample coverage, topic set, or measurement window can invalidate a comparison;
even matching source/model does not prove identical methodology. Historical provenance is not backfilled.

Perplexity's Sonar API is not the consumer search UI. The collector changed in October 2026;
do not infer a historical run's provider from its platform label or the merge date. If persisted
provider/model provenance is absent, mark it unknown and flag the possible methodology break
before comparing periods. Do not describe API answers as what a user saw in the consumer UI.

For a tracker readout, return the selected organization/site, run ID, observation date, source,
coverage/unknowns, and comparison window. Use the same organization and run IDs when comparing
ChatGPT MCP with this Grok wrapper. A matching endpoint configuration is not proof of customer
authentication or cross-client agreement.

Collection belongs to the backend. Do not keep a chat session polling to collect measurements.
Source code schedules visibility collection Mon/Wed/Fri at 06:00 UTC for eligible opted-in orgs;
report that as configured cadence, not a guaranteed next measurement. Verify runtime eligibility
and scheduler health before giving a next-run promise. Changes between observations are descriptive,
not evidence that a content edit caused the movement.

## The three visibility reads

**Rankings — where you stand.**

```
platform_get_visibility_rankings(platform="", include_all=False)
```

Brand mentions, average position, visibility %, and the competitor ranking table. `platform` filters
to `'chatgpt'` or `'perplexity'`; omit for all. `include_all=True` pulls in competitors the org
isn't explicitly tracking — useful for discovering who's actually beating them, as opposed to who
they *think* the competition is.

**Topics — where the gaps are.**

```
platform_get_visibility_topics(platform="", date_range_days=0)
```

Per-topic visibility score, mention count, and competitor ranking, with a per-prompt breakdown.
`date_range_days` accepts `7`, `30`, `90`, or `180`. This is the actionable one: an aggregate
visibility number tells you nothing about what to write, whereas a topic where competitors rank and
you're absent is a content brief.

**Sentiment — how you're described.**

```
platform_get_visibility_sentiment(platform="")
```

Positive and negative themes with occurrence counts. Negative themes recurring across assistants
usually trace back to something real and public — a pricing page that reads as opaque, a comparison
page a competitor wrote about you, a stale docs claim. Treat a recurring negative theme as a lead to
investigate, not a metric to optimize.

## AEO: the traffic side

```
platform_get_aeo_data(date_range='30d', content_filter='sitewide')
```

AI-referred sessions by platform, key events, and a page-level breakdown. `date_range` takes `'7d'`,
`'30d'`, `'90d'`, `'180d'`, `'12m'`, `'ytd'`. `content_filter='clickflow'` narrows to
ClickFlow-managed pages, which is how you separate "our program is working" from "the site gets AI
traffic."

**This requires a GA4 connection**, and it's slow (10–30 seconds). If GA4 isn't connected, say that
plainly — visibility data is not a substitute for traffic data, and you cannot infer one from the
other.

## Turning findings into work

Tracking is the default. Report the evidence and proposed next measurement first. Only use the
content actions below when the user requests content work; a tracking request does not authorize
generation, publishing, or paid collection.

Visibility findings are only worth producing if they change what gets written. The move is almost
always one of:

1. **A topic gap** — competitors rank, you're absent. Pull the per-prompt breakdown to see the
   actual phrasing, then hand it to `clickflow-content-pipeline` as a keyword plus a `prompt` steer.
2. **A page that ranks in search but not in AI answers** — usually a structure problem, not a
   content one. `platform_generate_faqs_for_url(target_url, delivery_mode='draft')` adds
   intent-driven Q&A and FAQPage schema, which is the highest-leverage single fix for extractability.
   3–8 minutes.
3. **A page assistants cite weakly** — `platform_optimize_page(content_url, target_keyword,
   optimization_context)` rewrites against what currently ranks. Put the visibility finding in
   `optimization_context` so the rewrite targets it.
4. **An authority gap on a page that deserves to rank** —
   `platform_boost_internal_links(target_url)`.

Cross-check against Search Console before acting: `platform_get_results_overview(date_range,
content_filter)` and `platform_get_page_performance(url, days=28)`. A page invisible to assistants
but climbing in search needs a different fix than one falling in both.

## Reporting honestly

- **Name the source and the window.** "Visibility 34% (ChatGPT, scan completed Aug 26)" is useful;
  "visibility is 34%" is not.
- **Assistants are non-deterministic.** A single scan is a sample. Movement between two consecutive
  scans is usually noise; prefer 90- or 180-day topic trends for anything a decision rests on.
- **Don't merge the two datasets into one score.** Scan-based visibility and GA4 AI traffic measure
  different things and disagree for legitimate reasons — being mentioned without being clicked is a
  real and common state, and it means something different from not being mentioned.
- **Report a missing integration as missing.** No GA4, no AEO numbers. Never estimate around it.
