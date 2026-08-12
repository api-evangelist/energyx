---
name: Monitor EnergyX open roles
description: Read the EnergyX careers surface from the public content API — resolve the position-area and position-location vocabularies, list open roles, and detect new or closed postings between runs.
api: openapi/energyx-careers-api-openapi.yml
operations: [listPositionAreas, listPositionLocations, listJobPositions, getJobPosition]
generated: '2026-08-12'
method: generated
---

# Monitor EnergyX open roles

12 open positions at capture, classified by two independent taxonomies: 8 functional areas and 14
locations. This is the cleanest hiring-signal surface EnergyX exposes and it is not advertised
anywhere as an API.

## Auth

None. Base URL `https://energyx.com/wp-json`.

## Steps

1. **Resolve both vocabularies** — `listPositionAreas`, `listPositionLocations`.
   `GET /wp/v2/position-area?per_page=100&_fields=id,slug,name,count`
   `GET /wp/v2/position-location?per_page=100&_fields=id,slug,name,count`
   Areas observed 2026-08-12: `all-positions` (12), `everything-engineering` (6),
   `corporate-positions` (5), `manufacturing-production` (3), `separation-technologies` (2),
   `creative-design` (0), `south-america-operations` (0), `battery-engineering` (0).
2. **List the roles** — `listJobPositions`.
   `GET /wp/v2/energyx-job-position?per_page=100&_fields=id,date,modified,slug,link,title,position-area,position-location`
   Both taxonomy fields are **arrays**; a role commonly carries three areas.
3. **Read a posting** — `getJobPosition` on an id, or add `content` to `_fields`.
4. **Filter** — `GET /wp/v2/energyx-job-position?position-area=<id>&position-location=<id>`.
5. **Diff between runs.** Snapshot the full id set each run (it is one request — there are only 12
   records). New ids are new postings; disappeared ids are closed roles. `modified_after` catches
   edits to existing postings but will NOT tell you a role was removed, so you need the id-set diff
   for closures.

## Rules

- **`all-positions` (term 63) is a catch-all**, attached to every record at capture. Filtering on
  it returns everything and tells you nothing — discriminate on the specific areas.
- **Zero-count terms are real, not errors.** `battery-engineering`, `creative-design` and
  `south-america-operations` all had zero open roles at capture; that absence is itself the signal.
- **Location terms are the deployment map.** 14 location terms against 12 roles means locations
  outnumber current postings — the vocabulary describes where EnergyX operates (Austin, Texarkana,
  Chile, Puerto Rico), not only where it is hiring today.
- **Never hard-code term ids**; resolve by `slug` every run.
- **Decode HTML entities** in `title.rendered` — e.g. `Control &#038; Instrumentation Engineer`.
- Self-throttle; no rate limits are published. Match errors on `code`
  (`errors/energyx-problem-types.yml`). A `rest_no_route` 404 on this collection means the theme
  stopped registering the post type, not a transient failure — see
  `lifecycle/energyx-lifecycle.yml`.
