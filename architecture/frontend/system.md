---
title: frontend-template — Mantine/Vite React template structure
tags:
  - frontend
  - react
  - mantine
  - vite
  - template
  - architecture
summary: '# frontend-template (mantine-vite-template)'
status: ready
updated: '2026-08-18'
---
# frontend-template (mantine-vite-template)

Repo: `/home/khanhth/projects/templates/frontend-template`. A production-oriented starter template (React 18 + Vite 5 + Mantine 7 + TypeScript 5), meant to save setup time on auth, API, layout and i18n boilerplate. Package manager: yarn 4 (Berry, `yarn@4.1.1`).

## Tech stack
- UI: Mantine core/dates/form/charts/notifications/modals, Tabler icons
- State: Redux Toolkit + redux-persist (PersistGate), react-redux hooks
- Routing: react-router-dom v6 (BrowserRouter)
- HTTP: axios (`ApiService`/`BaseService`)
- Forms/validation: mantine-form + yup (`mantine-form-yup-resolver`)
- i18n: i18next / react-i18next, locale files under `src/locales/lang/{en,es,tr}.json`
- Mock API: miragejs (`src/mock`), toggled via `appConfig.enableMock`
- Testing: vitest + @testing-library/react, `test-utils/render.tsx` custom render wrapper
- Storybook 8 for component docs (`*.story.tsx`)
- Lint/format: ESLint (airbnb + eslint-config-mantine), Stylelint, Prettier
- `test` script chains: typecheck → prettier → lint → vitest → build

## Directory layout (src/)
- `App.tsx` — composition root: MantineProvider → ModalsProvider → redux Provider → PersistGate → BrowserRouter → Notifications + Layout. Boots `mockServer()` when `appConfig.enableMock`.
- `theme.ts` — Mantine theme override.
- `@types/` — shared TS types: `auth.ts`, `layout.ts` (`LayoutTypes` enum), `navigation.ts`, `routes.tsx` (`Route`/`Routes` = lazy component + `authority: string[]`).
- `configs/`
  - `app.config.ts` — global `AppConfig`: `apiPrefix`, `authenticatedEntryPath` (`/dashboard`), `unAuthenticatedEntryPath` (`/sign-in`), `enableMock`, `locale`, `layoutType` (default `LayoutTypes.CollapsibleAppShell`).
  - `navigation.config/` — sidebar/nav item definitions.
  - `routes.config/` — `routes.config.ts` defines `publicRoutes` (spread of `authRoute`) and `protectedRoutes` (array of `{key, path, component: lazy(...), authority}`), plus `authRoute.tsx` and barrel `index.ts`.
- `route/` — routing/guards: `AppRoute.tsx` (wraps a route component, dispatches `setCurrentRouteKey` on location change for active-nav-on-refresh support), `ProtectedRoute.tsx`, `PublicRoute.tsx`, `AuthorityGuard.tsx`, `AuthorityCheck.tsx` (authority-based access control, backed by `useAuthority` hook).
- `components/`
  - `Layout/` — `Layout.tsx` (top-level layout switch), `AuthLayout.tsx`, `Views.tsx`, `LinksGroup.tsx` (+ css module). Multiple selectable layouts (collapsible/decked/simple app shells per README, driven by `appConfig.layoutType`).
  - `UserPopOver/` — user menu popover variants (`UserPopOver`, `CollapsedSideBarUserPopOver`, content components).
  - `ColorSchemeToggle/`, `LoadingScreen/`, `Welcome/` (has `.story.tsx` and `.test.tsx` as the example pattern for new components).
- `pages/`
  - `auth/SignIn.tsx` — sign-in page.
  - `examples/` — `Dashboard.tsx`, `Users.tsx`, `Manage.tsx`, `Pages.tsx`, `Files.tsx` — sample pages wired to `protectedRoutes`.
- `services/` — `BaseService.ts` (axios instance base), `ApiService.ts` (wraps BaseService), `auth/auth.service.ts` (auth API calls).
- `store/` — Redux setup: `storeSetup.ts`, `index.ts` (store + persistor + typed hooks re-export), `hook.ts` (`useAppSelector`/`useAppDispatch`), `rootReducer.ts` combines slices.
  - `slices/auth/` — `sessionSlice.ts`, `userInfoSlice.ts`, `userSlice.ts`, `constants.ts`, barrel `index.ts` exporting `AuthState`.
  - `slices/base/` — `commonSlice.ts` (`BaseState`).
  - `slices/locale/localeSlice.ts` — `LocaleState` (current locale).
  - `slices/theme/themeSlice.ts` — `ThemeState` (color scheme etc).
- `locales/` — `locales.ts`, `index.ts`, `lang/{en,es,tr}.json`.
- `mock/` — `mock.ts` (`mockServer()` bootstrap via miragejs), `fakeApi/authFakeApi.ts`, `data/authData.ts`.
- `utils/hooks/` — `useAuth.ts`, `useAuthority.ts`, `useLocale.ts`, `useQuery.ts`.
- `utils/deepParseJson.ts`.
- `constants/` — `api.constant.ts`, `app.constant.ts`.

## Path aliasing
Imports use `@/...` alias (via `vite-tsconfig-paths`) resolving to `src/` (e.g. `@/components/Layout/Layout`, `@/store`).

## Conventions worth reusing
- New route = add entry to `protectedRoutes`/`publicRoutes` in `src/configs/routes.config/routes.config.ts` with a `lazy()` import, then it's rendered through `AppRoute` which syncs `currentRouteKey` into redux on navigation (keeps sidebar nav active state correct across refresh — see commit "make collapsible app shells navigation active on page refresh").
- New component pattern: colocate `Component.tsx`, `Component.module.css`, `Component.story.tsx`, `Component.test.tsx` in the same folder (see `Welcome/`).
- Auth/authorization is `authority: string[]`-based per route, checked via `AuthorityGuard`/`AuthorityCheck`/`useAuthority`.
- Feature toggles like mock API live in `app.config.ts`, not env vars.
- Root `README.md` points to an external wiki (github.com/auronvila/mantine-template/wiki) for deeper docs — repo appears to be a fork/derivative of `auronvila/mantine-template`.
