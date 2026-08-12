---
name: Discover the EnergyX API surface from scratch
description: Re-derive the whole EnergyX content API from the host's own self-describing documents — the route index, the registered types and taxonomies, and the per-route OPTIONS schemas — so a client can be rebuilt when the undocumented surface shifts.
api: openapi/energyx-discovery-api-openapi.yml
operations: [getRouteIndex, listTypes, listTaxonomies, listStatuses, listUsers, getUser, search, getSeoHead, getOembed]
generated: '2026-08-12'
method: generated
---

# Discover the EnergyX API surface from scratch

EnergyX publishes no API documentation. Everything in this repository was derived from documents
the host serves about itself. This skill is how to redo that — necessary because the surface has no
versioning or deprecation policy and the interesting collections are theme-registered custom post
types that can vanish without notice.

## Auth

None. Base URL `https://energyx.com/wp-json`.

## Steps

1. **Read the route index** — `getRouteIndex`.
   `GET /wp-json/`
   Returns site name, the registered namespaces, and every route with its methods and args. 1,233
   routes across 36 namespaces at capture. Narrow it with `?namespace=wp/v2` — the full document is
   large.
   The `authentication` block here is the only place the host states its credential model:
   `application-passwords` only, authorization at
   `https://energyx.com/wp/wp-admin/authorize-application.php`.
2. **List the content types** — `listTypes`.
   `GET /wp/v2/types`
   26 registered types at capture. The `rest_base` field is the collection path and `rest_namespace`
   the namespace. The EnergyX-specific ones are `enx-press-release`, `energyx-in-the-news`,
   `energyx-indu-news`, `energyx-leadership`, `energyx-job-position`, `energyx-partner`,
   `energyx-video`, `resource-guide` and `energyx-megamenu`.
3. **List the taxonomies** — `listTaxonomies`.
   `GET /wp/v2/taxonomies`
   13 registered, with the `types` they classify. **Check for orphans**: `faq-type` is registered
   and readable but classifies `enx-faq`, a post type with **no REST route** (`/wp/v2/enx-faq`
   returns 404). A registered taxonomy does not prove its content type is reachable.
4. **Read each route's schema** — `OPTIONS <route>`.
   `curl -X OPTIONS https://energyx.com/wp-json/wp/v2/enx-press-release`
   Returns `{endpoints:[{methods, args}], schema:{title, properties}}` — the query parameters and
   the full JSON Schema for the resource. **This is the step that produces the OpenAPI.** Every
   spec in `openapi/` was generated from these documents; re-running it is a mechanical
   regeneration, not a rewrite.
5. **Check what anonymous access actually reaches.** A registered route is not a public route.
   `GET /wp/v2/settings` returns 401, the whole `wc/v3` namespace returns 401, and the response
   `Allow` header on a public collection reads `GET` only. Probe before modelling.
6. **Enumerate the searchable corpus** — `search` with `?search=` and read `X-WP-Total` (245).
7. **Two metadata side-doors** worth knowing: `getOembed`
   (`GET /oembed/1.0/embed?url=<energyx.com URL>`) for embed metadata, and `getSeoHead`
   (`GET /yoast/v1/get_head?url=<energyx.com URL>`) which returns the rendered head plus the parsed
   schema.org JSON-LD `@graph` for any page — structured data without scraping HTML.
8. **Public authors** — `listUsers` / `getUser`. 10 records; these are the bylines on posts, pages
   and media.

## Rules

- **A 200 in the route index is not a 200 for you.** The great majority of the 1,233 routes are
  admin, plugin-management or e-commerce-admin routes that answer 401 anonymously. See
  `mcp/energyx-tool-crosswalk.yml` → `host_routes_not_modelled` for the namespace-by-namespace map.
- **Do not read `/wp/v2/feedback` or `/gf/v2/entries`.** Those are submitted contact-form entries —
  third-party personal data. They are excluded on principle from every artifact in this repository.
- **Ignore the server headers.** The host returns deliberately falsified ones
  (`Server: Coffee Machine 1.2.4`, `X-Powered-By: Commodore 64 BASIC`). Infer nothing from them.
- **Ids are global, not per-type.** An id alone does not tell you the collection; carry `type` /
  `subtype` alongside it.
- Self-throttle; no rate limits are published. Match errors on `code`
  (`errors/energyx-problem-types.yml`).
