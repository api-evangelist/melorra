---
name: melorra-browse-jewellery-catalog
description: >-
  Search and page Melorra's public jewellery catalog by wear type, material, collection or price,
  using the faceted filter values the API returns rather than guessing filter strings.
api: Melorra Catalog API
base_url: https://services-catalog.melorra.com/api
operations:
  - listProducts
  - getApiRoot
generated: '2026-08-25'
method: generated
source: openapi/melorra-catalog-api-openapi.yml (operationIds verified against the spec)
---

# Browse the Melorra jewellery catalog

Melorra's catalog API is public and needs no key, token or account. Every request below is an
anonymous `GET`.

## 1. Confirm the endpoint surface

Call `getApiRoot` — `GET /product/` — to read the backend's own index of endpoints. It returns
`product`, `product_silver`, `products`, `products_silver`, `recommended` and `similar`.

Note the URLs it returns use the `http` scheme even though the service is served over `https`.
Rewrite them to `https` before following them.

## 2. Search the catalog

Call `listProducts` — `GET /product/products/`.

Useful parameters, all documented by Melorra in its own `/.well-known/api-catalog`:

- `wear_type` — `Earrings`, `Rings`, `Pendants`, `Necklaces`, `Bracelets`, `Bangles`
- `type` — material, e.g. `Gold`, `Gemstone`
- `set_name` — collection set name
- `trend` — fashion trend
- `page` — 1-based page number

**Do not use `special_price__range`.** It is documented, but Melorra's own documented example value
`10000,20000` returns HTTP 500. Filter on price client-side from `price` / `special_price` instead.

`trend=Classic` is also a documented example that matches zero products — do not treat an empty
result from `trend` as an error in your request.

## 3. Read the response carefully — the shape is not what the docs say

Melorra's api-catalog states "the actual data is always inside the `results` field". On this
endpoint `results` is an **object**, not an array:

```
count            total matching products
next / previous  page links
results
  base_image_path   join image paths to this
  base_video_path
  title
  breadcrumb
  products[]        <- the actual product records
filters            faceted filter values WITH counts
sorting            the legal sort keys
currency           always INR
conversion_rates   advisory USD / GBP / SGD / AED
```

So the products live at `results.products`, not at `results`.

## 4. Discover legal filter values instead of guessing

The `filters` block in every listing response enumerates each facet's real values and how many
products each one matches — `karat`, `special_price`, `gender`, `weight`, `wear_type`,
`base_colour`, `type`, `nav_menu`, `try_on`, `occasion`, `motif`.

Read the facet values from a first unfiltered call and use those exact strings. This is the only way
to enumerate them: there is no `/categories/`, `/sets/` or `/trends/` lookup endpoint.

## 5. Sort

The legal sort keys are published in the `sorting` block of the response, and also via
`OPTIONS /product/products/`: `sort+by+rank`, `sort+by+popular`, `sort+by+latest`,
`sort+by+discount`, `sort+by+price+high+to+low`, `sort+by+price+low+to+high`, `sort+by+quickship`.

## 6. Render images

Image entries are **relative paths**. Join them to `results.base_image_path` — a product record on
its own is not enough to render an image.

## Rules

- Read-only. The server advertises `Allow: GET, HEAD, OPTIONS`; there is nothing here to write,
  and therefore nothing to undo.
- No rate limits are published and no rate-limit headers are returned. Self-throttle. A full crawl
  of the 21,742-product catalog at 20 per page is roughly 1,100 requests against an endpoint with no
  published budget.
- Errors are Django REST Framework `{"detail": "..."}` on 404 but **HTML** on 500 — do not assume a
  JSON body on every response.
- Prices are integers in INR. The `conversion_rates` are advisory; the API accepts no currency
  parameter, so convert client-side.
