---
name: clickflow-seo
description: Orientation for working with a ClickFlow SEO workspace over MCP — connect, select the right organization, and find the right tool among the 51 available. Covers the read/write split, the human-approval gates around content generation and CMS publishing, async workflow polling, and what each dashboard surface (Results, AEO, Radar, Visibility) actually returns. Load this first for any ClickFlow request, then hand off to clickflow-content-pipeline, clickflow-decay-loop, or clickflow-ai-visibility for the specific job.
---

# Working with ClickFlow

ClickFlow is the system of record for an organization's SEO program: target keywords, a content
roadmap, drafts, CMS publishing, and the Search Console + AI-visibility measurement behind it. Your
job is orchestration and judgment; ClickFlow does the heavy lifting and remembers everything.

This skill is the map. For a specific job, use:

| Job | Skill |
|---|---|
| Research a keyword → draft → review → publish | `clickflow-content-pipeline` |
| Find and fix decaying pages on a standing weekly loop | `clickflow-decay-loop` |
| Brand visibility inside ChatGPT / Perplexity, AEO traffic | `clickflow-ai-visibility` |

## Before anything else: connect and select the org

1. **Verify the tools exist.** If the `platform_*` tools aren't in this session, STOP and tell the
   user to install and authorize the ClickFlow plugin. Never improvise results or fabricate data.
2. **Select the organization.** Call `platform_list_organizations()`, show the user the candidates,
   and have them confirm — then `platform_switch_organization(organization_id)` and echo the
   selected name back. Every subsequent call is scoped server-side to that org.

Acting against the wrong organization is the worst failure mode available here: it spends real
content quota and can change a live website. When more than one org exists, never guess.

3. **Load the brand voice** with `platform_get_brand_voice()` before generating or judging any copy.
   Everything ClickFlow drafts is brand-conditioned; if you write copy yourself, match it.

## The read/write split

Read-only tools (annotated `readOnlyHint`) are free to call freely to orient yourself. The write
tools fall into three tiers, and the tier determines how much permission you need:

- **Cheap and reversible** — `platform_add_target_keyword`, `platform_update_roadmap_item`,
  `platform_update_roadmap_item_status`, `platform_schedule_roadmap_item`,
  `platform_switch_organization`. Just do these when the request implies them.
- **Spends quota or takes 10–20 minutes** — `platform_create_roadmap_item(start_drafting=true)`,
  `platform_optimize_page`, `platform_boost_internal_links`, `platform_generate_faqs_for_url`.
  Each consumes a content-plan slot. Confirm scope and count before firing a batch, and check the
  remaining `content_slots` in `platform_get_loop_status()` so you size the batch to what's left
  instead of failing mid-run.
- **Changes a live website, or is destructive** — `platform_publish_to_cms`,
  `platform_unpublish_page`, `platform_revert_internal_links`. **Never call these without the
  user's explicit go-ahead on that specific item.** Leave drafts and hand over review links; let
  the human press publish.

Never execute deletions, redirects, `noindex`, or page consolidations. Recommend them with evidence
and let the human act outside the loop.

## Async workflows

Content generation is asynchronous. Any tool whose docstring says "returns a workflow_id" starts a
job that takes minutes, not seconds:

1. Start it — you get back `{workflow_id, ...}`.
2. Poll `workflows_get_status(workflow_id)` for `status`, `progress`, and `current_action`. Pass
   `output_format='markdown'` if you want the content as Markdown rather than HTML.
3. A workflow can **pause for human review**. When it does, `status` reflects that and `review.type`
   / `review.data` describe what's being asked. Inspect those, present the decision to the user, and
   submit their answer with `workflows_resume(workflow_id, decision)`. Don't answer a review gate on
   the user's behalf — keyword and internal-link reviews are the approval gates.
4. Poll at a sane cadence. Drafting runs 10–20 minutes, internal-link boosts 5–15, FAQ generation
   3–8. Tell the user the expected wait rather than polling in a tight loop.

`platform_list_workflows(status, limit)` shows what's already in flight — check it before starting
work on a URL, because a stale suggestion will happily re-offer a page that's already been drafted.

## Review links are minted server-side

Every finished draft has a `review_url`. **Never construct one yourself** — a guessed `/edit/...`
URL opens an empty document. Collect it, in this order, from:

1. the response of the tool that started the work,
2. the completed workflow's `output.review_url` from `workflows_get_status`,
3. `platform_get_roadmap_item(roadmap_item_id).review_url`.

If a completed draft genuinely has no `review_url`, say so and hand the user the `workflow_id`.

## What the measurement surfaces actually return

| Tool | Source | Notes |
|---|---|---|
| `platform_get_results_overview(date_range, content_filter)` | Google Search Console | Clicks, impressions, CTR, position + page-level data vs. previous period. `content_filter='clickflow'` narrows to ClickFlow-managed pages. Slow: 10–30s. |
| `platform_get_page_performance(url)` | Google Search Console | One page's trend — use it to confirm a decay candidate. |
| `platform_get_aeo_data(date_range, content_filter)` | Google Analytics 4 | AI-referred traffic by assistant. **Requires a GA4 connection.** Slow: 10–30s. |
| `platform_get_visibility_*` | ClickFlow scans | Brand position inside AI assistants — see `clickflow-ai-visibility`. |
| `platform_get_radar_sitemaps` / `platform_get_radar_pages` | Tracked sitemaps | Site inventory: discovered URLs, titles, keyword data, tracking status. |

`date_range` accepts `'7d'`, `'30d'`, `'90d'`, `'180d'`, `'12m'`, `'ytd'`.

Search Console is the dependency for results, page performance, and all decay detection. If it
isn't connected, say so plainly — don't substitute a guess.

## One-shot generators

The `agents_*` tools are fast, stateless helpers that don't touch the roadmap: 20–60 seconds, no
quota slot, useful mid-conversation.

`agents_optimize_blog_title`, `agents_optimize_meta_description`, `agents_generate_faqs`,
`agents_research_company`, `agents_generate_image`, `agents_analyze_competitors`,
`agents_analyze_search_intent`.

Reach for these when the user wants an answer rather than a change to their workspace.

## Failure handling

ClickFlow fails explicitly rather than returning plausible defaults, and you should pass that
through. If a tool returns an `error`, report it — don't retry blindly, don't fall back to
inventing the data, and don't paper over a missing integration. A `{"skipped": ...}` result is not
a success: check for it rather than treating any truthy response as a measurement.
