---
title: product
tags:
  - contracts
summary: '# Product ("Gói cước") — API contract (features api 017 / web 019)'
status: draft
links:
  - repos/api.md
  - repos/web.md
updated: '2026-11-18'
---
# Product ("Gói cước") — API contract (features api 017 / web 019)

Page `/business-rule/products`, `menu_key` = `product`, permissions `product.view|create|update|publish` (granted manually on CMS before deploy; legacy unauthenticated write endpoints were removed).

## Endpoints
- Screen reads (static, ≥2 segments so legacy `GET /products/:id` cannot shadow): `GET /products/manage/list` (filters `business_type` default 1, `service_vmp_id`, `service_vmp_type`, `status` exact — `0` is valid, `product_code`, `product_name`, `page`, `limit` ≤200 → `{items,total}`), `GET /products/manage/options`, `GET /products/manage/:id`.
- Writes: `POST /products` and `PATCH /products/:id` (multipart, image field `image_smarttv`, PATCH sends changed fields only); `PATCH /products/:id/status` (compare-and-set 0↔1, 2→0, 409 on race); `/hidden-device` (`hidden_device_list_v2` CSV); `/plugin` (`product_plugin_list_last` CSV — the column admin2 uses, not CMS1's `product_plugin_list`).
- Duplicate `product_code` → 422 "Mã gói cước đã tồn tại" (unique index). Non-OTT (`service_vmp_id`≠1) requires an existing `clip_id`; OTT forces `clip_id` null. Create with CDN image failure → 502 whose message carries the created product ID.
- `/product-module` (`PRODUCT_ID, MODULE_ID, AREA_CODE_LIST CSV, STATUS` 0 Lock / 1 View / 2 View with locked icon; `PATCH /product-module/status` toggles 1↔0), `/product-area` (`product_id, area_code, user_team_id`; names resolved server-side), `GET|PUT /products/:id/services` (PUT = desired state per `service_type`, diff in one transaction), `GET /product-config/get-by-product-id?product_id`, `PUT /product-config` (`product_id, visible_product_ids`, keeps existing sort, appends new), `PUT /product-config/sort`, `POST /products/publish` (`mytv_type` 1 → ott host, else internal host; returns per-target results).
- Image CDN follows CMS1 exactly (`synCDN(..., 'products', ..., pimg_path)` stored into `image_smarttv`).

## Traps
- Envelope `{status,result,message,data,errors}`; web `ApiService` returns the envelope, payload is `.data`.
- Index migration `product/migration/261118_products_indexes.sql` must run before relying on 422-by-unique-index and p99 target (<100 ms list).
- Web keeps wizard `/products/package-configuration` (step 1 reuses `ProductForm`; steps 2–5 are UserTeam/Service modules).
