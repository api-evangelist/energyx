---
name: Read the EnergyX merchandise catalog
description: Read the EnergyX WooCommerce storefront through the anonymous Store API — products, prices in minor units, stock and categories — and understand why the wp/v2 projection of the same products is not a substitute.
api: openapi/energyx-store-api-openapi.yml
operations: [listStoreProductCategories, listStoreProducts, getStoreProduct, listProducts, getProduct]
generated: '2026-08-12'
method: generated
---

# Read the EnergyX merchandise catalog

energyx.com/shop/ sells branded apparel and accessories through WooCommerce. 15 products, 4
categories, USD. It is **merchandise, not lithium products and not API access** — nothing here
prices the company's chemicals or its API.

The same 15 products are exposed twice, with the same ids and different field sets:

| Surface | Route | Carries |
|---|---|---|
| WooCommerce Store API | `/wc/store/v1/products` | prices, currency, stock, variations, image sets, inlined categories |
| WordPress core | `/wp/v2/product` | title, slug, content, featured_media, category **ids** — **no price** |

Use the Store API. Use the core projection only when you need the WordPress-side fields.

## Auth

None for the Store API. The WooCommerce **admin** API (`/wc/v3/*`) on the same host returns
**401 `rest_forbidden`** to an anonymous caller and there is no public key issuance path — treat it
as unavailable, not as "needs a key you can get".

## Steps

1. **Read the categories** — `listStoreProductCategories`.
   `GET /wc/store/v1/products/categories?per_page=100`
   4 categories at capture, each with a `count`.
2. **Read the catalog** — `listStoreProducts`.
   `GET /wc/store/v1/products?per_page=100`
   `X-WP-Total` is 15, so one page covers it.
3. **Read one product** — `getStoreProduct`.
   `GET /wc/store/v1/products/<id>`
4. **Cross-reference the core record** if you need `content.rendered` or `featured_media` —
   `getProduct` at `GET /wp/v2/product/<id>` with the **same id**.

## Rules

- **Prices are strings of MINOR units.** `{"price": "3500", "currency_minor_unit": 2,
  "currency_code": "USD"}` is **$35.00**. Divide by `10 ** currency_minor_unit`. Never parse the
  string as a decimal and never assume 2 decimal places without reading the field.
- **Use `currency_code`, `currency_symbol`, `currency_prefix`/`currency_suffix` and the separator
  fields for display** — they are all on the `prices` object, so do not hard-code `$`.
- **Store API error codes are namespaced differently.** WooCommerce returns
  `woocommerce_rest_product_invalid_id`, not `rest_post_invalid_id`. Key your handling on the full
  code string — see `errors/energyx-problem-types.yml`.
- **`is_in_stock` is the field to trust**, not a derived reading of quantity.
- **Do not touch `/wc/store/v1/cart` or `/wc/store/v1/checkout`.** They are session-mutating and
  deliberately excluded from this read-only profile
  (`mcp/energyx-tool-crosswalk.yml` → `deliberate_exclusions`).
- **Decode HTML entities** in `name` — `T-Shirt &#8211; From Brine to Battery™`.
- Self-throttle; no rate limits are published.
