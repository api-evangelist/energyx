---
name: Track EnergyX company news
description: Pull EnergyX press releases, third-party coverage and blog posts from the public content API, deduplicate them into one timeline, and poll for change without re-reading everything.
api: openapi/energyx-press-api-openapi.yml
operations: [listPressReleases, getPressRelease, listInTheNews, listIndustryNews, listPosts, search]
generated: '2026-08-12'
method: generated
---

# Track EnergyX company news

EnergyX splits its news across four collections that no single endpoint joins. Read all four.

| Collection | Route | Records (2026-08-12) | What it is |
|---|---|---|---|
| Press releases | `/wp/v2/enx-press-release` | 39 | First-party announcements |
| In the news | `/wp/v2/energyx-in-the-news` | 275 | Third-party coverage of EnergyX |
| Industry news | `/wp/v2/energyx-indu-news` | 5 | Curated lithium/energy sector news |
| Blog | `/wp/v2/posts` | 69 | EnergyX-authored articles |

## Auth

None. `https://energyx.com/wp-json` is anonymously readable. Do not send an Authorization header.

## Steps

1. **Read the newest press releases** — `listPressReleases`.
   `GET /wp/v2/enx-press-release?per_page=20&_fields=id,date,modified,slug,link,title,excerpt`
   Results are newest-first. Read `X-WP-Total` (39) and `X-WP-TotalPages` from the response headers
   to size the walk.
2. **Repeat for the other three** — `listInTheNews`, `listIndustryNews`, `listPosts`, same parameter
   shape. The in-the-news archive is the big one at 275 records; `per_page` is hard-capped at 100
   (see `errors/energyx-problem-types.yml`, `rest_invalid_param`), so it takes three pages.
3. **Merge on `id`.** Ids are unique across every post type on this host, so a plain union
   deduplicates correctly. Keep `subtype`-equivalent provenance by tagging each record with the
   collection it came from — the record itself carries `"type"` values that do not distinguish
   press releases from posts in every projection.
4. **Fetch the full body only when you need it** — `getPressRelease` on a single id, or add
   `content` to `_fields` on the list call. An unfiltered record is large; the list projection with
   `_fields` is not.
5. **Poll for change with `modified_after`, not by re-reading.** There is no ETag, no
   Last-Modified and no Cache-Control on this surface (`X-Cache-Status: BYPASS`), so a conditional
   request is not available. Store the highest `modified` timestamp you have seen and issue
   `GET /wp/v2/enx-press-release?modified_after=<ISO8601>&per_page=100`. An empty array is the
   "nothing changed" answer.
6. **Free-text across everything** — `search`.
   `GET /wp/v2/search?search=<term>&per_page=20` returns lightweight `{id, title, url, type,
   subtype}` pointers across all 245 searchable objects. Dispatch on `subtype` to know which
   collection to resolve against, or follow `_links.self.href`, which is cheaper.

## Rules

- **Decode HTML entities.** `title.rendered` is rendered HTML — `Control &#038; Instrumentation`
  means `Control & Instrumentation`. `content.rendered` and `excerpt.rendered` are full HTML.
- **`featured_media: 0` means no image.** Never call `GET /wp/v2/media/0`.
- **Self-throttle.** No rate limit is published and no `RateLimit-*` or `Retry-After` header is
  returned (`rate-limits/energyx-rate-limits.yml`). Keep concurrency low and space out page walks.
- **Match errors on `code`, never on `message`** — see `errors/energyx-problem-types.yml`.
- **Nothing here is retryable-by-idempotency-key.** Every operation is a read; there is no
  idempotency contract on this host and none is needed.
- **This surface is unmanaged.** These are theme-registered custom post types with no deprecation
  policy and no status page. Handle a sudden `rest_no_route` 404 as a schema change, not a
  transient error, and re-read `GET /wp-json/` to see what still exists.
