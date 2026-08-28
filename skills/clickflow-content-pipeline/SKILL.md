---
name: clickflow-content-pipeline
description: Take a keyword or a content idea through ClickFlow end to end — validate search intent and competition, create a roadmap item, generate a brand-aligned draft, handle the workflow's human review gates, then publish to WordPress, Webflow, Shopify, HubSpot, Sanity, or Strapi once a human approves. Use when the user wants new content written, a draft reviewed or revised, an existing page optimized or given FAQs, or something published to their CMS.
---

# ClickFlow content pipeline

New content, from keyword to live page. Read `clickflow-seo` first if you haven't — org selection
and the workflow-polling rules live there and apply throughout.

**To fix an existing page rather than write a new one, stop here.** Use `platform_optimize_page` (or
`clickflow-decay-loop` for the standing version). Creating a new roadmap item for a URL you already
rank with splits your own impressions and wastes a content slot.

## 1. Validate before you spend a slot

Drafting consumes a content-plan slot, so qualify the keyword first. All of these are read-only:

- `platform_analyze_keyword(keyword, location='United States', language='en')` — volume,
  difficulty, and the surrounding data.
- `agents_analyze_search_intent(...)` — what the searcher actually wants, which decides the format.
- `platform_list_target_keywords(limit)` — is this already tracked, or a near-duplicate of
  something tracked? If a sibling keyword is already covered, act on one page, not two.
- `platform_get_radar_pages(limit=200)` — does a URL on the site already map to this intent? If yes,
  optimize that page instead.
- `agents_analyze_competitors(...)` / `platform_list_competitors()` — what's ranking and in what
  format.

Then check capacity: `platform_get_loop_status()` reports `content_slots`. Size the batch to what
remains.

If the keyword is worth tracking regardless, add it with
`platform_add_target_keyword(keyword, volume, difficulty, intent)` — `intent` is one of
`informational`, `commercial`, `transactional`, `navigational`.

## 2. Create the roadmap item

```
platform_create_roadmap_item(
  keyword,
  prompt="",                  # extra steer: angle, must-cover points, audience
  strategy="Article",
  scheduled_date="",          # ISO, e.g. '2026-09-15'
  start_drafting=False,
  image_generation="",
  delivery_mode="draft",
  cms_connection_id="",
)
```

- `strategy` — `Article`, `Listicle`, `How-to guide`, `Case study`, `News post`, `Review`,
  `Comparison`, `FAQ`. Pick from what actually ranks for the query, not from preference.
- `image_generation` — `stock_photos`, `diagrams`, `illustrations`, `watercolor`. Omit for none.
- `start_drafting=True` fires the drafting workflow immediately and returns a `workflow_id`.
  **This is the point where quota is spent** — confirm with the user before setting it on a batch.
- Keep `delivery_mode='draft'`. Only use `'cms'` (with a `cms_connection_id`) when the user has
  explicitly asked for auto-delivery, and understand that it writes to their CMS.

Sanity-check the keyword before you pass it: a malformed keyword becomes the draft's title.
Numbering and query-syntax junk (`36.abm platform experts`) must be repaired or dropped first.

## 3. Ride the workflow, including its review gates

Poll `workflows_get_status(workflow_id)`. Drafting takes 10–20 minutes — tell the user that rather
than polling tightly.

When the workflow pauses for review, `review.type` and `review.data` say what's being asked. Present
it and submit the user's answer via `workflows_resume(workflow_id, decision)`. **These gates are the
approval mechanism** — a keyword review or an internal-link review is where a human curates what
ships. Don't answer them yourself to keep things moving.

## 4. Hand over the draft — don't grade it silently

Collect the server-minted `review_url` (never build one) and post it as a clickable link.

Before you do, spot-check the draft and flag — don't fix — anything the reviewer shouldn't have to
discover on their own:

- product plugs or CTAs belonging to some other company,
- sentences repeated verbatim across drafts from the same batch,
- statistics with no source,
- unfilled placeholders,
- a title or meta description that doesn't match the page.

For the last one, `agents_optimize_blog_title` and `agents_optimize_meta_description` are fast and
free; propose the improvement and let the user take it. `platform_update_roadmap_item(roadmap_item_id,
title, meta_description, slug)` applies it — only provided fields change.

To write or replace a body wholesale, `platform_save_draft(roadmap_item_id, title, html,
meta_description, thumbnail_url)` sets status to `drafted`.

Move status with `platform_update_roadmap_item_status(roadmap_item_id, status)` — valid values are
`ready for review`, `approved`, `changes requested`, `drafted`, `synced`, `archived`.

## 5. Publish only on an explicit go-ahead

```
platform_list_cms_connections()          # id, type, status — never credentials
platform_publish_to_cms(roadmap_item_id, cms_connection_id="", force=False)
```

What to know before calling it:

- **It updates in place** when the item maps to an existing CMS resource, so it won't silently
  duplicate a post.
- **It errors rather than guessing** in two cases: re-publishing an item whose CMS copy has changed
  since ClickFlow last wrote it, and being unable to check the target URL against the CMS. Both are
  overridable with `force=True` — **only after you surface the specific risk and the user agrees.**
  `force` on case 1 overwrites their edits; on case 2 it creates a new draft, so check for
  duplicates afterwards.
- Pass `cms_connection_id` explicitly when the org has more than one connection.

`platform_unpublish_page(published_content_id)` is a **soft delete in ClickFlow tracking only** — it
does not remove the post from WordPress or anywhere else. Say that out loud when you use it, get
confirmation first, and tell the user they still have to delete it in the CMS.

## Adjacent moves worth offering

- `platform_generate_faqs_for_url(target_url, delivery_mode='draft')` — intent-driven Q&A plus
  FAQPage schema for an existing page. 3–8 minutes. Good AEO leverage on a page that already ranks.
- `platform_boost_internal_links(target_url, manual_keywords="")` — finds pages on the site to link
  *from* into a target page. 5–15 minutes. The link review is the approval gate: reject forced
  anchors and off-topic source pages. `platform_revert_internal_links(workflow_id)` undoes the
  auto-applied ones — confirm before reverting.
- `platform_schedule_roadmap_item(roadmap_item_id, date)` — set or move a publication date.

## After it's live

`platform_list_published_content()` confirms what ClickFlow considers live.
`platform_get_page_performance(url, days=28)` reads back Search Console clicks, impressions, CTR,
and position against the prior equal window. Nothing meaningful shows up in under a week or two —
don't read a two-day result as a verdict.

If the user wants a standing loop that measures every refresh and compounds what it learns, that's
`clickflow-decay-loop`.
