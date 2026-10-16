---
title: content cate
tags:
  - contracts
summary: '# Content category contract (VOD / Movie / Music) — feature 011'
status: ready
links:
  - contracts/content-media-upload.md
  - contracts/content-edit.md
updated: '2026-10-16'
---
# Content category contract (VOD / Movie / Music) — feature 011

Shared create / detail / update / status flow for `CONTENT_CATE` (VOD), `MOVIE_CATE` (Movie), `MUSIC_VIDEO_CATE` (Music).
API core: `api:src/modules/content/shared/content-cate/` (`ContentCateService`, field catalogue, `ContentCateBodyPipe`,
`ContentCateStatusPipe`, tree helpers). Web: `web:src/components/shared/contentCate/` (`CategoryForm`, `CategoryList`,
`cateTree`, `cateFormValues`). Paths and `@RouteInfo` names are unchanged from before 011.

## Routes

| | VOD (`menu vod-category`) | Movie (`movies-category`) | Music (`music-clip-cate`) |
|---|---|---|---|
| list | `GET content/content-category`, `GET content/v2/content-category`, `GET content/content-category/list-parent/:CATE_ID` | `GET content/cate-movie/get-cates-by-parent/:type_id` | `GET content/music-cate` |
| detail | `GET content/content-category/detail/:CATE_ID` | `GET content/cate-movie/get-cates-by-parent/:type_id/:cate_id` (TYPE_ID must match) | `GET content/music-cate/:cateId` |
| create | `POST content/content-category` | `POST content/cate-movie` | `POST content/music-cate` |
| update | `PATCH content/content-category/:CATE_ID` | `PATCH content/cate-movie/:cate_id` | `PATCH content/music-cate/:cateId` |
| status | `PUT content/content-category/status/:CATE_ID` (`vod-category.update-status`) | `PUT content/cate-movie/status/:cate_id` (reuses `movies-category.update`) | `PATCH content/music-cate/:cateId/status` (`music-clip-cate.update-status`) |
| images | `POST content/media-upload/vod-cate/{create,update}` | `.../movie-cate/{create,update}` | `.../music-cate/{create,update}` |

- **List responses are unchanged byte-for-byte** (legacy `BaseController` envelope; other screens call them: PhoneRule,
  CateBar, VotingEvent, Tournament, MusicClip Info). SQL is now a flat parameterised SELECT; the VOD tree
  (`CONTENT_NAME_FILTER`, `location`, `depth` as string) is built in TS (`legacyTreeRows`). Verified 128/128 identical vs
  the old code on stage data (2026-10-16). Quirks kept: VOD v1 `CATE_STATUS` filters roots only; v2 without
  `CATE_STATUS` returns only rows whose status is NULL; v2 picks the table by TYPE_ID (incl. `CEESHOW_CATE` for 31).
- Detail / create / update / status use the ADR 0001 envelope (`ContentEditResponseInterceptor`) and return the DB row.
- Old multipart `PUT` update paths return **409** "Trang đã có phiên bản mới, vui lòng tải lại trang" (stale bundles);
  remove after 1–2 releases. Status with an empty body also returns 409.

## Body (JSON, column names kept)

- Create = full body; update = PATCH with changed fields only (absent = unchanged). Validation from the catalogue
  (`content-cate-fields.ts`): unknown / read-only / other-kind / create-only-on-update field → 400 (Vietnamese).
- Clear rules: strings / images `null` → `''`; `TYPE_ID_LINK`, `POSTER_LAYOUT`, `CATE_NODE_TYPE` `null` → NULL; id lists
  (`CATE_ID_LINK`, `HIDDEN_DEVICE_LIST_V2`) sent as int arrays, stored CSV, cleared with `[]`; required fields can't be cleared.
- Required on create: VOD `CATE_NAME`, `CATE_NAME_EN`, `CATE_ORDER`, `TYPE_ID`; Movie + `CATE_NAME_ADMIN`; Music `CATE_NAME`,
  `CATE_NAME_EN` (TYPE_ID forced to 3, not sendable). `CATE_NODE_TYPE` is a free int (legacy rows hold 3 / NULL).
- Server-owned: `CATE_ID`, `CATE_CODE`, `HIDDEN_DEVICE_LIST` (v1, derived from v2 for VOD/Movie; Music has no v1 column),
  `CATE_NAME_FULL` (Movie TYPE 82 = `CATE_NAME`), `LINK_URL` (VOD LiveScore), `USER_*`, dates.
- Images: CDN path from the cate media-upload kind + token in `MEDIA_TOKENS` (tokens are not interchangeable with content kinds).
- Status body: `{ CATE_STATUS?, B2B_CATE_STATUS? }` (B2B only VOD / Music), values 0 / 1, explicit set (no toggle).

## Tree rules

- `CATE_CODE` = root `'0'`, child `'0_<root>_…_<parent>'`, rebuilt from the real parent chain (legacy stale codes ignored).
- Changing `PARENT_ID` recomputes the whole subtree in one transaction with one batched `UPDATE … CASE`; audit entry lists
  `DESCENDANT_CATE_CODES` old → new. Self / descendant as parent → 400; parent must exist with the same TYPE_ID; any code
  longer than 50 chars (varchar(50)) → 400 for the whole move.
- Web parent picker: VOD / Movie list only active categories (legacy behaviour kept, owner 2026-10-16); Music lists all.

## Testing notes

- E2E: `web/e2e/support/contentCate.ts` (`defineCateSuite`) + `{vod,movie,music}-cate.spec.ts` write only `[E2E-011] …` rows
  on the shared stage DB; delete them afterwards with `api/.ai/features/011-content-cate-standardization/baseline/cleanup-e2e-cates.js <TABLE> preview|delete <ids>`.
