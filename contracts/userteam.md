---
title: userteam
tags:
  - contracts
summary: >-
  # User Team ("User Team" / nhóm thuê bao) — API contract (feature api 020 /
  web next)
status: ready
links:
  - repos/api.md
  - repos/web.md
  - contracts/product.md
  - contracts/service.md
updated: '2026-11-26'
---
# User Team ("User Team" / nhóm thuê bao) — API contract (feature api 020 / web next)

Page `/business-rule/user-team`, `menu_key` = `user-team` (member DB `cmssmart_admin_menu`, menu_id 49616). Permissions `user-team.view|create|update|delete` (grant manually before deploy). Full contract + CMS1 mapping: `api/docs/user-team.md`. Migration from CMS1 with documented fixes; no stored procedure of this module is called any more except legacy `sync-service`.

## Envelope
`{status,result,message,data,errors}`; errors are Vietnamese `HttpException` messages (404 not found, 409 conflict, 422 validation/business, 503 lock-wait/member-guard timeout).

## Endpoints (prefix `user-team`)
- Catalog: `GET manage/list?keyword&service_vmp_id&page&limit≤100` → `{items,total,page,limit}` (keyword on name/description **AND** vmp; order id ASC; 30/page). `GET manage/:id` → user_team row. `POST /` `{service_vmp_id,name≤100,description≤200}` (all required; Product wizard uses this) → row. `PATCH /:id` (only sent fields; extra fields stripped) → row. `DELETE /:id` (`user-team.delete`) → `{id,deleted:{channels,services,subjects,logs,team}}`; 409 when members exist or `product_area` references (message lists product names); 503 when the member check times out (20 s). `POST /:id/duplicate {name}` (`create`) → new row (copies channels with all columns, services, subjects incl. TYPE_ID).
- Services per content type: `GET :id/services?type` → `{user_team:{id,name},type,services:[{service_id,service_name,checked}]}`; `PUT :id/services {type, service_ids:number[]}` → `{changed,added,removed}` ([] = remove all; unknown/wrong-type id → 422).
- Subjects: `GET :id/subjects` → `{user_team,items:[{id,subject_id,TYPE_ID,subject_name,subject_parent,subject_type,createdate,username}]}`; `DELETE :id/subjects/:rowId`; `GET :id/subjects/tree?type` → `{type,type_name,source,nodes:[{id,name,parent_id,active,checked,children}]}` (source MOVIE_CATE for 2/21/25, MUSIC_VIDEO_CATE for 3, else CONTENT_CATE; assigned inactive cates included with active=false); `PUT :id/subjects {type_id, cate_ids}` → `{count}`; `POST :id/subjects/sync {target:'net'|'flexi'}` → `{target,ok,results:[{host,ok,status?,error?}]}` (never throws on host failure).
- Channels: `GET :id/channels?name&tvod&tstv` (all-digit name = channel number; 0 is a valid filter; all statuses) → `{user_team,items:[{id,channel_id,name,channel_number,channel_status,status,tvod,tstv,sort_order,area_code_list,thumbnail_smartphone,thumbnail_tv}]}` — `sort_order` = team's own order, `channel_number` = channel.sort_order. `GET :id/channels/options` → `{tvod:number[],tstv:number[]}`. `GET :id/channels/assignable` → `{channels:[{id,name,sort_order,tvod,tstv,checked,team_status:1|0|null}],areas:[{area_code,name}]}`. `PUT :id/channels {channel_ids,tvod,tstv,area_codes?}` → `{inserted,reactivated,removed}`. `PATCH :id/channels/status {ids,status:0|1}` → `{updated}`. `PATCH :id/channels/:rowId {tvod?,tstv?,sort_order?,area_codes?}` → row. `PATCH :id/channels/:rowId/status` (toggle) → `{id,status}`. `DELETE :id/channels/:rowId` → `{id}`.
- Compare/copy: `GET manage/compare?one&two` → `{one,two,rows:[{channel_id,name,channel_number,one?:{id,tvod,tstv},two?:{…},differs}]}` (active rows only). `POST :id/channels/copy {channel_id,tvod,tstv}` → `{skipped}`. `POST :id/channels/copy-all {from_user_team_id}` → `{removed,inserted}` (**overwrites** target, copies area_code_list).
- Auth-only (other screens, paths unchanged): `GET /user-team` (Product select, `limit=-1`), `POST channel {channel_id,list_userteam_id:'1,2'}` → `{inserted,skipped}`, `POST channel/delete-all {list_userteam_id:'rowId,rowId'}` → `{deleted}`, `DELETE channel/:id` → `{result,id}`, `PUT sync-userteam/:channel_id` → `{channel_id,updated}`, `PUT sync-service/:channel_id` (legacy).
- Deprecated (old UserTeam screen only; web must stop using, then API removes): `get-detail/:id`, `get-user-team-subject`, `delete-user-team-subject/:id`, `get-user-team-service`, `update-user-team-service`, `get-user-team-channel`, `get-tvod-options/:id`, `get-tstv-options/:id`, `update-user-team-channel[-status]`, `delete-user-team-channel/:id`. Removed: `all-list-orderby-name`, `channel/add-multi`.

## Traps for the frontend
- Assign = final set: unselected channels are hard-deleted (including rows of channels disabled in `channel`, like CMS1); selected inactive rows are re-activated; existing rows keep id/tvod/tstv/area/sort_order. New channels get min(channel, form) tvod/tstv and the form's areas. Warn before save.
- Every channel-set change recomputes `channel.PRODUCT_VIEW_LIST` server-side (no client action).
- Do not send the channel number as `sort_order` (old dialog bug); 0 is valid for tvod/tstv/sort_order. Old delete URL lacked a `/`.
- No `GET /user-team/:id` — that path belongs to product-legacy (`/user-team/:product_id`); use `manage/:id`.
- Deleting a team without members can take ~15 s (full scan of member_mytv until DBA adds an index).
- Deploy order: grant `user-team.*` → API → web. Keep RTK hook names (`useGetUserTeamQuery`, `useCreateUserTeamMutation`) and type `IUserTeam` used by Product.

## Updates after code review (2026-11-26)
- `POST /user-team/channel` now writes `sort_order = 0` (team's own order) instead of the channel number.
- `POST /:id/duplicate` returns the new user_team row (has `id`); copies `code, service_vmp_id, description, tvod, tstv` (not `channel_changed`).
- `GET :id/channels/assignable` pre-checks rows that are currently **inactive** (`team_status = 0`); saving the assign form unchanged re-activates them. The web must show the state and let the user uncheck before saving.
- Net sync hosts come from optional env `USER_TEAM_SUBJECT_HOSTS` (comma list; empty = the 4 default MyTV Net hosts).
