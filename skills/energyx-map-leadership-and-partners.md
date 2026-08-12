---
name: Map the EnergyX leadership team and partner network
description: Build the EnergyX org and partner graph from the public content API by resolving the leadership-type and partner-type vocabularies first, then filtering the rosters by term.
api: openapi/energyx-leadership-api-openapi.yml
operations: [listLeadershipTypes, listLeadership, getLeader, listPartnerTypes, listPartners, getPartner, getMediaItem]
generated: '2026-08-12'
method: generated
---

# Map the EnergyX leadership team and partner network

33 leadership records across 7 types, 8 partners across 2 types. Both are classified by
site-specific taxonomies whose term ids you must resolve before you can filter.

## Auth

None. Base URL `https://energyx.com/wp-json`.

## Steps

1. **Resolve the leadership vocabulary first** — `listLeadershipTypes`.
   `GET /wp/v2/leadership-type?per_page=100&_fields=id,slug,name,count`
   Observed 2026-08-12: `executive-leadership` (7), `science` (8), `operations` (6),
   `marketing` (0), `board-of-directors` (8), `science-advisory-board` (2), `advisory-board` (3).
   **Never hard-code these ids.** They are site-scoped and change on a site rebuild; resolve by
   `slug` every run.
2. **Pull the roster** — `listLeadership`.
   `GET /wp/v2/energyx-leadership?per_page=100&_fields=id,slug,link,title,featured_media,leadership-type`
   `leadership-type` is an **array** — a person can be both an executive and a board member. Do not
   assume one type per person.
3. **Filter by term when you want one group** —
   `GET /wp/v2/energyx-leadership?leadership-type=<id>&per_page=100`.
4. **Get the bio** — `getLeader` on a single id, or add `content` to `_fields` on the list call.
5. **Get the headshot** — `featured_media` is an attachment id; resolve with `getMediaItem`, or
   avoid the second round trip entirely by adding `_embed` to the list request, which inlines the
   attachment under `_embedded["wp:featuredmedia"]`.
6. **Do the same for partners** — `listPartnerTypes` then `listPartners`.
   `GET /wp/v2/energyx-partner?per_page=100&_fields=id,slug,link,title,featured_media,partner-type`
   Observed records include Convergence, Critical Minerals Institute and Morken Group. The
   `featured_media` on a partner record is the partner logo.

## Rules

- **Resolve vocabularies before filtering.** A hard-coded term id is the single most likely thing
  to break on this surface; the taxonomy endpoints are cheap and authoritative.
- **`count: 0` is real.** `marketing` had zero leadership records at capture. Do not treat an empty
  filter result as an error.
- **Use `_embed`, not N+1.** One embedded list request replaces up to 33 media lookups.
- **Decode HTML entities** in `title.rendered` (`Dr. Farid Vaezi`, `Chris Alex` are clean; others
  carry `&#8217;` and similar).
- **This is public personal data published by the company about its own staff.** Read it; do not
  enrich it with data from elsewhere, and do not use it to build contact lists — see the API
  Evangelist enrichment PII posture.
- Self-throttle; no rate limits are published. Match errors on `code`
  (`errors/energyx-problem-types.yml`).
