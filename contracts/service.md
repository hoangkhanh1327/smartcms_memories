---
title: service
tags:
  - contracts
summary: >-
  # Service ("Service" / gói dịch vụ nội dung) — API contract (feature api 019 /
  web 021)
status: ready
links:
  - repos/api.md
  - repos/web.md
  - contracts/product-plugin.md
updated: '2026-11-25'
---
# Service ("Service" / gói dịch vụ nội dung) — API contract (feature api 019 / web 021)

Page `/business-rule/service`, `menu_key` = `service` (member DB `cmssmart_admin_menu`, menu_id 49619), permissions `service.view|create|update` (no delete; grant manually before deploy). Full contract + CMS1 mapping: `api/docs/service.md`. Scope is **migration: CMS1 semantics kept** except documented fixes. No stored procedure of this module is called any more (services_get_lst, content_service_channel_update/copy/copy_all/get_by_service ported to TypeScript).

## Endpoints (prefix `service`, envelope `{status,result,message,data,errors}`)
- Catalog: `GET manage/list` (`service_code` LIKE, `type`, `provider_id`, `paytv_id`, `page`, `limit`≤100 → `{items,total}` with `type_name`, `paytv_name`), `GET manage/:id`, `POST /`, `PATCH /:id` (only sent fields; same URL as legacy so the old web keeps working; whitelist strips extra fields).
- PATCH side-effects run **only when values change**, in one transaction with a 30 s lock-wait cap (1205 → 503): `price_ott`/`provider_id` → `MOVIE.MOVIE_PRICE/PROVIDER_ID` + `CONTENT.CONTENT_PRICE/PROVIDER_ID` WHERE SERVICE_ID; `service_name` → `product_service` + `user_team_service`. Server derives `price_tv = price_sd`, `price_mobile = price_ott` (edit).
- Channels (service_type 1 only, else 422): `GET :id/channels?name&tvod&tstv` (all-digit name = sort_order; -1 = all), `GET :id/channels/options`, `GET :id/channels/assignable` (`checked` flag), `PUT :id/channels {channel_ids}` = final set → `{inserted,reactivated,removed}`, `PATCH :id/channels/status {content_ids,status}`, `PATCH :id/channels/:contentId {tvod?,tstv?}`, `PATCH :id/channels/:contentId/status` (toggle), `DELETE :id/channels/by-channel/:channelId` (all duplicate rows of the pair).
- Compare/copy: `GET compare?one&two` → `{one,two,rows[{channel_id,name,sort_order,one|null,two|null}],summary}`; `POST :id/channels/copy {channel_id,tvod,tstv}` (`{skipped}`), `POST :id/channels/copy-all {from_service_id}` (**overwrites** target).
- Areas: `GET|POST :id/channels/:channelId/areas`, `PATCH|DELETE …/areas/:areaCode`, `PATCH …/areas/:areaCode/status` (409 duplicate, area_code must exist in `area`).
- Contents (read + detach only): `GET :id/contents/summary` → `{movie,content,'music-video'}`, `GET :id/contents?source&keyword&page`, `DELETE :id/contents/:source/:contentId` (SERVICE_ID = 0).
- Legacy kept auth-only: `GET /service`, `GET /service/:id`, `/service/service-pay`, `/service/get-*`, `/service/channel*` (web content/Channel/Info). Tokens `IServicesRepository`/`IServicesService` + old file paths (re-export shims) unchanged.

## Traps for the frontend
- Assign keeps CMS1 semantics: saving re-activates every selected inactive channel and hard-deletes unselected ones (warn the user). Existing TVOD/TSTV kept, new channels get 0/0.
- Bulk status is by `content_id` (CMS1 bug sent channel_id). Data has duplicate (service, channel) rows (133 pairs) — rows are keyed by `content_id`.
- `GET /service/compare` is a static segment; Fastify resolves it before legacy `GET /service/:id`.
- Deploy order: grant permissions → API → web 021.
