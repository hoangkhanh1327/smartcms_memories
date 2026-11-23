---
title: product plugin
tags:
  - contracts
summary: >-
  # Product Plugin ("Product Plugin" / IPTV package) — API contract (feature api
  018 / web 020)
status: draft
links:
  - repos/api.md
  - repos/web.md
  - contracts/product.md
updated: '2026-11-23'
---
# Product Plugin ("Product Plugin" / IPTV package) — API contract (feature api 018 / web 020)

Page `/business-rule/product-plugin`, `menu_key` = `product-plugin` (member DB `cmssmart_admin_menu`, menu_id 49613), permissions `product-plugin.view|create|update|delete` (granted manually on CMS before deploy). Legacy endpoints stay auth-only (~40 files in other modules use the 3 string tokens `IProductPluginService/Repository/ServicesService` — never break them; `IProductPluginChannelService/Repository` were removed 2026-11-23, nothing outside used them). Data-flow doc: `api/docs/product-plugin.md`. Scope is **migration only: CMS1 logic is kept as-is**; `ProductPluginV2`/Promotion (B2C) are out of scope and live in `plugin-b2c`.

## Endpoints (all under `product-plugin`, envelope `{status,result,message,data,errors}`)
- Reads (≥2 segments, cannot be shadowed by legacy `:id` routes): `GET manage/list` (`business_type` default 1, `service_vmp_id`, `product_service`, `status` exact — 0 valid, `product_code`, `product_name`, `page`, `limit`≤200 → `{items,total}`), `GET manage/:id`.
- Writes: `POST /` and `PATCH /:id` (JSON now, NOT multipart; `product_bundle_logo` comes from `content/media-upload/product-plugin` signed upload + `MEDIA_TOKENS`; PATCH still accepts `isactive` until the web switches); `PATCH :id/status` (compare-and-set 0↔1, 409 on race); `PATCH :id/hidden-device` (`hidden_device_list_v2` CSV, v1 derived); `DELETE :id` (409 while channel/service rows exist).
- Duplicate `product_code` → 422 **only on create or when the code changes** (DB already has `Sport` twice: products 61, 137 — so no unique index).
- Services: `GET :id/services?service_type` → `{selected, services}`; `PUT :id/services` body `{service_type?, service_ids[]}` = desired state, one DELETE + one INSERT in a transaction; `service_name/service_type` come from `services` (client values ignored); existing duplicate rows collapsed.
- Channels: `GET :id/channels?name&tvod&tstv` (all-digit name = `channel.sort_order`; tvod/tstv=0 valid; shows `isactive=0` rows too), `GET :id/channels/options` → `{tvod[],tstv[]}`; `PUT :id/channels` body `{channel_ids[], tvod, tstv}`; per-row `PATCH channels/:rowId` (`tvod,tstv,sort_order`), `PATCH channels/:rowId/status`, `DELETE channels/:rowId`.

## Traps
- **Assign channels = port of SP `product_plugin_channel_update`, bug-for-bug.** All rows of the product are set `isactive=0` (the old admin2 toggled instead — fixed), existing `(product, channel)` rows are only re-enabled (their tvod/tstv are NOT changed by the form), new rows are inserted with channel name/tvod/tstv from `channel`: `tvod=MIN(channel.tvod, form.tvod)`, and the SP typo is kept: `if channel.tstv > form.tvod then tstv = form.tstv` (so form 3/120 raises a 60-minute channel to 120). Empty `channel_ids` turns every channel off. Unknown channel id → 422 (the SP would insert NULLs). Table has no unique `(product_id, channel_id)` and already holds duplicates (product 1, channels 742/745). Fixing the typo changes data → separate decision, only after CMS1 stops using the SP (CMS1 still calls it; never drop/alter the SP).
- `product_plugin_service` also has duplicate rows; `service_type` is an int column (entity fixed from string).
- Description/short_desc are only trimmed (no editor sanitising like CMS1 `processContentEditor`).
- Migration `product-plugin/migration/261123_product_plugin_indexes.sql` (plain indexes only) is run by the developer; list/channel p99 measured 27–39 ms (mostly network) on tiny tables.
- Old endpoints are gone: legacy `POST/PATCH/DELETE /product-plugin[/:id]` and the old channel/service endpoints (`get-product-plugin-channel`, `assign-product-plugin-channel`, `get-tvod/tstv-options`, `get/update-product-plugin-service`, …). Web 020 (merged into `feat/upgrade_cms_112026` 2026-11-23) uses only the new routes. The api no longer calls any SP in `product`/`product-plugin`. Remaining legacy read routes: `GET /product-plugin`, `/get-detail/:id`, `/list`, `/list-all`, `/list-condition-by-desc`.
