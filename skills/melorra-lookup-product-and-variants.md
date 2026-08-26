---
name: melorra-lookup-product-and-variants
description: >-
  Look up a specific Melorra product by SKU and find its karat options, ring sizes, weights,
  stock availability and shipping lead times — including the workaround for the per-SKU detail
  route that does not work.
api: Melorra Catalog API
base_url: https://services-catalog.melorra.com/api
operations:
  - getProductBySku
  - listProductDetails
  - listSimilarProducts
generated: '2026-08-25'
method: generated
source: openapi/melorra-catalog-api-openapi.yml (operationIds verified against the spec)
---

# Look up a Melorra product and its variants

## 1. Get the product summary by SKU

Call `getProductBySku` — `GET /product/products/{sku}/`, e.g. `/product/products/232484/`.

This returns the summary record: `sku`, `ext_product_id`, `code`, `design_code`, `product_title`,
`type`, `wear_type`, `set_name`, `trend`, `price`, `special_price`, `discount_message`,
`quick_ship`, `express`, `try_on`, `images` and `melorraProductUrl`, plus `currency` and
`conversion_rates`.

A SKU that does not exist returns `404` with `{"detail": "Not found."}`.

## 2. Understand the product codes

- `design_code` — the design family, e.g. `C22CC117F`. Shared across every karat and size.
- `code` — the full buyable variant, e.g. `C22CC117F-XX-12-109Y00`: design, then a size segment,
  then a karat segment.
- `sku` / `ext_product_id` — integer ids. `sku` is the one the path takes.

If you need to identify something a customer can actually buy, you need the variant `code`, not the
`sku`.

## 3. Get variant pricing, sizes and stock — the awkward part

The karat/size matrix lives only on the richer projection returned by `listProductDetails` —
`GET /product/product/`.

**`GET /product/product/{sku}/` returns HTTP 500.** There is no working per-SKU route on this
projection. To get variant detail for one product you must page `GET /product/product/` (10,000
records at 20 per page) and match on `pricing.product_code` or
`product_data.product_specifications.design_code`.

Plan for this: if you need variant data for more than a couple of products, page the projection once
and index it locally rather than searching it per lookup.

Each detail record carries:

- `pricing.karat` — a map keyed `9 Karat` … `24 Karat`, each with `title`, `weight`,
  `is_available` and its own `product_code`. Unavailable karats appear with `null` title and weight
  and `is_available: false`.
- `pricing.size[]` — `diameter`, `circumference`, `weight_change_percent`, the variant
  `product_code`, and per-karat `quickship_k09_count` … `express_k22_count` stock counts.
- `pricing.dimension` and `pricing.diamond_caratage` — free text, e.g. `0.0 carat SI IJ`.
- `product_data.shipping_data` — `manufacturing_days`, `hallmarking_days`, `processing_days`,
  `transit_days`, `lead_time` and delivery date ranges for standard, express and quick-ship.
- `product_data.collection` and `product_data.seo`.

Note `weight_change_percent` on a size: gold weight — and therefore price — changes with ring size.
Do not quote the summary `price` as the price of a specific size.

## 4. Find similar products

Call `listSimilarProducts` — `GET /product/similar/?sku={sku}`.

This endpoint has a **different response shape again**. There is no `results` member: `products[]`,
`base_image_path` and `base_video_path` sit at the top level next to `count`, `next` and `previous`.

## 5. Silver is a separate endpoint

Silver products are not reachable by a material filter. Use `listSilverProducts` —
`GET /product/products_silver/` (165 records) — or the silver detail projection at
`/product/product_silver/`.

## Rules

- Anonymous and read-only. No key, no account, nothing to reverse.
- Three endpoints, three different pagination shapes. Never reuse a response parser across
  `/product/products/`, `/product/product/` and `/product/similar/`.
- Image paths are relative and must be joined to the base path returned alongside them.
- No versioning, no deprecation policy and no changelog are published. Treat every field as subject
  to silent change and validate defensively.
