---
name: Harvest the EnergyX research and media library
description: Pull the EnergyX resource guides, video library and media attachments from the public content API, grouped by their vocabularies, with the pagination discipline the 2,639-record media collection requires.
api: openapi/energyx-resource-guides-api-openapi.yml
operations: [listResourceGuideCategories, listResourceGuides, listVideos, getVideo, listMedia, getMediaItem, listCategories, listPosts]
generated: '2026-08-12'
method: generated
---

# Harvest the EnergyX research and media library

Three collections make up the EnergyX educational corpus: 14 long-form resource guides, 61 video
records, and 2,639 media attachments. The guides are the highest-value text; the media library is
the collection that will break a naive client.

## Auth

None. Base URL `https://energyx.com/wp-json`.

## Steps

1. **Resolve the guide vocabulary** — `listResourceGuideCategories`.
   `GET /wp/v2/resource-guide-category?per_page=100&_fields=id,slug,name,count`
   Observed 2026-08-12: Cleantech (11), Sustainability (7), Battery (4), Lithium (4), Company (2).
   Counts sum above 14 because guides carry several categories.
2. **Pull the guides** — `listResourceGuides`.
   `GET /wp/v2/resource-guide?per_page=100&_fields=id,date,modified,slug,link,title,excerpt,resource-guide-category`
   Add `content` when you want the full text; these are long documents, so fetch bodies in a second
   pass rather than in the list call.
3. **Pull the videos** — `listVideos`.
   `GET /wp/v2/energyx-video?per_page=100&_fields=id,date,slug,link,title,featured_media`
   No taxonomy is registered against this post type, so there is nothing to filter on — sort by
   `date` instead. Records include conference panels and TV appearances.
4. **Walk the media library carefully** — `listMedia`.
   `GET /wp/v2/media?per_page=100&page=<n>&_fields=id,date,slug,link,title,mime_type,media_type,media_details,post`
   2,639 records at `per_page=100` is 27 pages. `per_page=999` returns **400
   `rest_invalid_param`** — the cap is 100 and there is no bulk export.
   Filter before you walk: `?media_type=image`, `?mime_type=application/pdf`,
   `?search=<term>`, or `?parent=<post id>` to get only the assets attached to one record.
5. **Cross-reference blog coverage** — `listCategories` then
   `GET /wp/v2/posts?categories=<id>` to pair guides with the blog articles on the same subject
   (`lithium` 58 posts, `cleantech` 44, `sustainability` 43, `battery` 23, `company` 21).

## Rules

- **`media_details` carries the generated size variants** — `sizes.thumbnail`, `sizes.medium`,
  `sizes.large` and their URLs. Pick a size rather than downloading the full-resolution original
  2,639 times.
- **`per_page` is capped at 100.** Paginate; check `X-WP-Total` and `X-WP-TotalPages` first so you
  know the cost before you start.
- **No conditional requests.** No ETag or Last-Modified is returned and `X-Cache-Status` is
  `BYPASS`, so every page is an origin read. Use `modified_after` for incremental runs and cache
  aggressively on your own side.
- **`post` on an attachment is polymorphic** — the parent may be any post type, not only a blog
  post. Do not assume `GET /wp/v2/posts/<post>` will resolve it.
- **Respect the noindex signal.** Every response carries `X-Robots-Tag: noindex`. The data is
  public and machine-readable, but the provider is asking that it not be republished as indexable
  content.
- Self-throttle; no rate limits are published. Match errors on `code`
  (`errors/energyx-problem-types.yml`).
