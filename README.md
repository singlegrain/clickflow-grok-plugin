# ClickFlow plugin for Grok Build

[ClickFlow](https://www.clickflow.com) is an SEO and content-marketing platform: it researches
keywords, plans a content roadmap, drafts brand-aligned articles, publishes them to your CMS, and
measures what happened in Google Search Console and across AI assistants.

This plugin connects Grok Build to ClickFlow's hosted MCP server, so an agent can run that whole
loop against your real ClickFlow workspace.

## Install

```
/plugin install clickflow
```

On first use you'll be sent through a browser OAuth flow to authorize your ClickFlow account. No
API key to copy, and nothing to configure — see [Authentication](#authentication).

## What it ships

| Component | Detail |
|---|---|
| MCP server | `clickflow` — hosted, HTTP transport, OAuth 2.1 |
| Skills | `clickflow-seo`, `clickflow-content-pipeline`, `clickflow-decay-loop`, `clickflow-ai-visibility` |
| Hooks | none |
| Commands / agents | none |

The plugin adds no scripts, no install steps, and no local processes. It is a single remote MCP
server declaration plus Markdown skills.

## Requirements

- A ClickFlow account with at least one organization — [start here](https://www.clickflow.com).
- A connected **Google Search Console** property, for anything that reads or measures performance
  (results overview, page performance, content-decay detection, refresh readbacks).
- A connected **CMS** (WordPress, Webflow, Shopify, HubSpot, Sanity, or Strapi), only
  if you want the agent to publish rather than leave drafts for review.

## Authentication

The MCP server is protected by OAuth 2.1, issued by ClickFlow's Clerk tenant. Discovery is
standards-based, so no client credentials are pre-provisioned or embedded in this repo:

1. An unauthenticated request to `https://api.clickflow.com/v1/mcp` returns `401` with
   `WWW-Authenticate: Bearer resource_metadata="https://api.clickflow.com/.well-known/oauth-protected-resource/v1/mcp"`.
2. That protected-resource metadata names the authorization server: `https://clerk.clickflow.com`.
3. The client registers dynamically at `https://clerk.clickflow.com/oauth/register`
   ([RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591)) and completes an authorization-code
   flow with PKCE (`S256`).

Scopes requested: `openid`, `profile`, `email`, `offline_access`.

Your token authorizes access to **your own** ClickFlow organizations and nothing else. Every tool
call is scoped server-side to the organization currently selected via `platform_switch_organization`.

For unattended use (a cron job, a headless runner) ClickFlow also accepts a `cf_ak_` API key as a
bearer token, since an interactive OAuth flow can't complete without a browser. Create one in the
ClickFlow dashboard.

## Network endpoints

This plugin talks to exactly two hosts, both operated by ClickFlow:

| Host | Purpose |
|---|---|
| `api.clickflow.com` | The MCP server and its OAuth protected-resource metadata |
| `clerk.clickflow.com` | OAuth authorization server: registration, authorize, token, JWKS |

No telemetry is collected by the plugin. It reads no local files, no environment variables, and no
credentials from disk.

## Tools

51 tools in three namespaces. Read-only tools are annotated `readOnlyHint`; the two destructive
tools are annotated `destructiveHint` so a client can gate them.

**`platform_*` — your workspace (42 tools)**

- *Organizations & brand:* `list_organizations`, `switch_organization`, `get_brand_voice`
- *Research:* `analyze_keyword`, `list_target_keywords`, `add_target_keyword`, `list_competitors`
- *Roadmap & drafts:* `list_roadmap_items`, `get_roadmap_item`, `create_roadmap_item`,
  `update_roadmap_item`, `update_roadmap_item_status`, `schedule_roadmap_item`, `save_draft`
- *Publishing:* `list_cms_connections`, `publish_to_cms`, `list_published_content`,
  `unpublish_page` *(destructive)*
- *On-page & internal links:* `optimize_page`, `generate_faqs_for_url`, `boost_internal_links`,
  `revert_internal_links` *(destructive)*
- *Results:* `get_results_overview`, `get_page_performance`, `get_aeo_data`, `get_radar_pages`,
  `get_radar_sitemaps`
- *AI visibility:* `get_visibility_rankings`, `get_visibility_topics`, `get_visibility_sentiment`,
  `get_visibility_status`
- *Decay & refresh loop:* `detect_content_decay`, `list_suggestions`, `approve_suggestion`,
  `dismiss_suggestion`, `record_refresh_outcome`, `list_refresh_outcomes`,
  `measure_refresh_outcome`, `get_loop_status`, `get_loop_learnings`, `record_loop_learning`
- *Workflows:* `list_workflows`

**`workflows_*` — long-running jobs (2 tools)**

`get_status`, `resume` — content generation runs asynchronously; poll status and resume at a
human-review checkpoint.

**`agents_*` — one-shot generators (7 tools)**

`optimize_blog_title`, `optimize_meta_description`, `generate_faqs`, `research_company`,
`generate_image`, `analyze_competitors`, `analyze_search_intent`.

### MCP prompts

The server also exposes three prompts, if your client lists them: `content_decay_loop`,
`refresh_review`, `weekly_readout`.

## Safety model

Content generation and publishing spend real quota and can change a live website, so the skills in
this plugin enforce human gates rather than leaving them to the model:

- Drafts are the default. Publishing to a CMS requires explicit user approval.
- Deletions, redirects, `noindex`, and page consolidations are **recommended, never executed**.
- Every draft returns a server-minted `review_url`; the skills never construct one.
- Per-cycle caps keep review load and content-plan consumption bounded.

## Support

- Docs: <https://docs.clickflow.com>
- Issues with this plugin: <https://github.com/singlegrain/clickflow-grok-plugin/issues>

## License

MIT — see [LICENSE](LICENSE). ClickFlow is a product of Single Grain.
