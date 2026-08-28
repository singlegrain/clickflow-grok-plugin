---
name: clickflow-ai-visibility
description: Measure and improve how a brand shows up inside AI assistants — position versus competitors in ChatGPT and Perplexity, topic-by-topic visibility, sentiment themes in how assistants describe the brand, and the AI-referred traffic actually landing on the site (AEO). Use when the user asks about AI visibility, answer-engine optimization, whether ChatGPT recommends them, how they compare to competitors in AI answers, or where AI traffic is coming from.
---

# AI visibility and answer-engine optimization

Two different questions, two different data sources. Keep them apart — conflating them produces
confident nonsense.

| Question | Tools | Source |
|---|---|---|
| Do assistants mention and recommend us, and how do we rank against competitors? | `platform_get_visibility_*` | ClickFlow's own scans of ChatGPT / Perplexity |
| Is AI-referred traffic actually arriving on the site? | `platform_get_aeo_data` | Google Analytics 4 |

Read `clickflow-seo` first for org selection. Everything here is read-only, so orient freely.

## Check the scan first

`platform_get_visibility_status()` returns the latest scan's state — `running`, `completed`, or
`failed` — with a timestamp. Call it before reporting numbers. Reporting a stale or partial scan as
current is the main way this goes wrong. If it's `running`, say so and give the timestamp of the
data you're describing; if `failed`, report that rather than presenting whatever's cached as fresh.

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
