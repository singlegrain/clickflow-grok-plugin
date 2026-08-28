# Refresh rubric — what a shippable refresh must contain

ClickFlow's refresh workflow (`platform_optimize_page` / `platform_approve_suggestion`) already generates against this rubric. Use it to (a) sanity-check a draft before handing it to a human, and (b) explain to the user why a refresh will or won't move rankings. The `refresh_review` MCP prompt automates this check.

## Why "reword a few sentences" fails (Google freshness patent US20120209838A1)

Google fingerprints a page on first crawl and compares on every recrawl. Changes below a weighted threshold are **ignored** — the update does nothing. Sentence-level tweaks and date bumps are in the ignored bucket. To count as a real refresh, ship **high-weight** changes.

| Change | Weight |
|---|---|
| Updated title | High (single highest) |
| New substantive section (new vocabulary/term vector) | Medium-High |
| New outbound/internal link + descriptive anchor | High |
| Updated anchor text on an existing link | High |
| Reworded copy | Low → often ignored |
| Date / lastmod bump only | None |

**A refresh must ship at least THREE high-weight changes.** If a draft only rewords, send it back.

## SERP-format fit

Read what actually ranks for the keyword and match the dominant format (listicle / how-to / ultimate guide / comparison). If the title promises "ideas / ways / examples", the body must deliver at least **5** concrete, numbered items. A format mismatch means the page needs a rewrite, not additions.

## AEO structures (answer-engine visibility)

- A direct-answer paragraph immediately after each major H2.
- One FAQ block, 3–5 question/answer pairs.
- Named-entity, specific claims (formats, product names, numbers) — not vague filler.

## Links

- 3–5 relevant internal links; prefer service/commercial pages where apt. Every anchor's text must match its destination's actual topic — flag anchor↔target mismatches.
- External links only where they genuinely help (the SG house style often omits them for competitive topics).

## Humanization

Strip AI tells: em-dash overuse, rule-of-three cadence, inflated symbolism, AEO boilerplate. Exactly **one** closing CTA (no duplicate after the FAQ).

## Positioning guardrail

Never position the product for a capability it doesn't have (e.g. "X is payment collection for tournaments, not tournament-management software"). Reuse the org's brand/product profile.

## Methodology doc

Every refresh also produces a methodology HTML (surfaced in the workflow output / editor) explaining the SERP winners, coverage gaps, and which patent changes were applied — a useful artifact to share with the reviewer.
