# Expo App Overview

This repo ships a **Code & Theory React Native template** built on **Expo SDK 55**. It includes navigation, theming, TanStack Query, i18n, design tokens, API layer scaffolding, and multi-flavor dev/QA/prod configuration.

Related docs in this repo:

- [CT Code Metrics](./CT-METRICS.md) — AI attribution and template-vs-custom tracking
- [Project Reporting](./PROJECT-REPORTING.md) — daily Jira → ROVO → Slack EOD reports

---

## Table of contents

1. [Tech stack](#tech-stack)
2. [Project structure](#project-structure)
3. [Architecture](#architecture)
4. [Flavors and environment](#flavors-and-environment)
5. [Local development](#local-development)
6. [Native builds and prebuild](#native-builds-and-prebuild)
7. [Navigation](#navigation)
8. [Theming and design tokens](#theming-and-design-tokens)
9. [Data layer](#data-layer)
10. [Internationalization](#internationalization)
11. [Shared utilities](#shared-utilities)
12. [Quality gates](#quality-gates)
13. [CI integration notes](#ci-integration-notes)

---

## Tech stack

| Layer | Technology |
|---|---|
| Framework | Expo ~55, React Native 0.83, React 19 |
| Language | TypeScript ~5.9 |
| Navigation | React Navigation 7 (drawer, tabs, native stack) |
| Server state | TanStack Query v5 |
| HTTP | Axios |
| Storage | react-native-mmkv |
| i18n | i18next + react-i18next + expo-localization |
| UI primitives | Custom `AppText`, `AppButton`, `AppList`, `AppTextInput` |
| Bottom sheets | @gorhom/bottom-sheet |
| Lists | @shopify/flash-list |
| Dev client | expo-dev-client (not Expo Go) |
| Node | ≥ 20 |

---

## Project structure

```
├── App.tsx                    # Root component — provider tree
├── index.ts                   # Entry point
├── app.config.ts              # Expo config, flavors, native IDs
├── eas.json                   # EAS Build profiles (dev / preview / production)
├── package.json               # Scripts and dependencies
├── tokens/                    # Figma design token source
│   ├── design.tokens          # Raw Figma export
│   └── semantic-colors.json   # Semantic color mapping
├── scripts/
│   └── generate-design-tokens.cjs
└── src/
    ├── api/                   # Axios client, endpoints, error types
    ├── components/            # AppText, AppButton, AppList, TopBar, …
    ├── config/                # env.ts, appDisplayName
    ├── connectivity/          # NetInfo wrapper
    ├── designSystem/          # Generated tokens + useDesignSystem hook
    ├── navigation/            # Navigators, linking, types
    ├── query/                 # QueryClient, hooks, query keys
    ├── screens/               # Feature screens (home, explore, profile, …)
    ├── services/              # API service modules
    ├── settings/              # Remote flags hooks
    ├── theme/                 # ThemeContext, AppColors, useThemedStyles
    ├── third-party/
    │   ├── i18n/              # i18next setup, locales, language switching
    │   └── localstorage/      # MMKV wrapper
    └── utils/
        ├── emptyErrorStates/  # EmptyStateView, ErrorStateView
        ├── errorBoundary/     # App-wide error boundary
        └── loading/           # Loading overlay, skeletons, inline loading
```

All application source lives under `src/` per Code & Theory engineering standards.

---

## Architecture

### Provider tree

`App.tsx` wraps the app in this order (outer → inner):

```
GestureHandlerRootView
  └── AppThemeProvider          # light / dark / system theme
        └── BottomSheetModalProvider
              └── SafeAreaProvider
                    └── QueryProvider       # TanStack Query
                          └── ConnectivityProvider
                                └── LoadingProvider
                                      └── AppRootErrorBoundary
                                            └── ThemedNavigationContainer
```

### Layer responsibilities

| Layer | Responsibility |
|---|---|
| **Screens** | UI composition, navigation params, local UI state |
| **Services** | API calls, data transformation |
| **Query hooks** | Caching, loading/error states via TanStack Query |
| **API client** | Axios instance, base URL from env, interceptors |
| **Theme / design system** | Colors, spacing, typography tokens |
| **Navigation** | Route definitions, deep linking, typed params |

---

## Flavors and environment

Three flavors share one codebase with different native package IDs and API URLs.

| Flavor | `APP_ENV` | Display name suffix | Android package suffix |
|---|---|---|---|
| Development | `development` | Dev | `.dev` |
| Staging (QA) | `staging` | QA | `.qa` |
| Production | `production` | (none) | (none) |

### How flavor is resolved

1. **`APP_ENV`** shell variable (set by npm scripts via `cross-env`)
2. **`EAS_BUILD_PROFILE`** on EAS builds (`development` → dev, `preview` → staging, `production` → prod)
3. Defaults to `development` if unset

**Important:** Do not put `APP_ENV` in `.env` files — it can shadow npm scripts and force the wrong flavor.

### API base URL

Resolved in `app.config.ts` → embedded in `extra.apiBaseUrl` → read at runtime via `expo-constants` in `src/config/env.ts`.

| Flavor | Default API URL |
|---|---|
| Development | `https://dev-api.example.com/` |
| Staging | `https://qa-api.example.com/` |
| Production | `https://api.example.com/` |

On EAS builds, `EXPO_PUBLIC_API_BASE_URL` from `eas.json` overrides. Locally, `.env.development` can set the dev URL without leaking into QA/prod runs.

### Native identifiers

Configured in `app.config.ts` (scaffold placeholders replaced by the project generator):

- **iOS bundle ID:** `com.codeandtheory.testprojectreportingkitcursoragent[.dev|.qa]`
- **Android package:** same pattern
- **Deep link scheme:** `exporn` (aligned with `src/navigation/linking.ts`)

---

## Local development

### Prerequisites

- Node.js ≥ 20
- npm (this repo ships `package-lock.json` — use `npm ci` in CI)
- For device/simulator builds: Xcode (iOS) and/or Android Studio after prebuild

### Install and start

```bash
npm install
npm start          # development flavor (Metro)
```

Flavor-specific Metro:

```bash
npm run start:dev   # development
npm run start:qa    # staging
npm run start:prod  # production
```

### Run on a simulator or device

This template uses **expo-dev-client**, not Expo Go. You need a native build first:

```bash
npm run ios:dev      # prebuild + run iOS (development)
npm run android:dev  # prebuild + run Android (development)
```

After the first prebuild for a flavor, use `*:run` scripts to skip prebuild when switching builds within the same flavor:

```bash
npm run ios:dev:run
npm run android:dev:run
```

QA and production variants follow the same pattern (`ios:qa`, `android:prod`, etc.).

---

## Native builds and prebuild

### When to prebuild

Run `expo prebuild --clean` with the correct `APP_ENV` when:

- First setting up native directories
- Switching flavors (package ID changes)
- Adding native modules or config plugins

The npm scripts wire prebuild automatically for `ios:*` and `android:*` commands.

### Committed native dirs

For CI (Fastlane, Gradle, CocoaPods), `android/` and `ios/` should be generated, committed, and kept in sync after prebuild. The RN DevOps pipeline expects them to exist.

### Config plugins

| Plugin | Purpose |
|---|---|
| `./plugins/withAndroidPip.js` | Enables Picture-in-Picture on Android MainActivity |

iOS PiP and background audio modes are configured in `app.config.ts` (`UIBackgroundModes`).

### EAS Build profiles

`eas.json` defines three profiles:

| Profile | Flavor | Distribution | Android output |
|---|---|---|---|
| `development` | dev | internal | APK (debug) |
| `preview` | staging | internal | APK |
| `production` | prod | store | AAB |

```bash
eas build --profile development --platform ios
eas build --profile production --platform android
```

---

## Navigation

Built with React Navigation 7 in a nested hierarchy:

```
RootNavigator (native stack)
├── Main (drawer)
│   ├── TabRoot (bottom tabs)
│   │   ├── HomeTab → HomeStack
│   │   ├── ExploreTab → ExploreStack
│   │   ├── ProfileTab → ProfileStack
│   │   └── PostsTab → PostStack
│   ├── Settings
│   ├── About
│   └── CarouselCatalog
├── ExampleModal (presentation: modal)
├── TransparentModal (presentation: transparentModal)
└── FullScreenModal (presentation: fullScreenModal)
```

### Key files

| File | Purpose |
|---|---|
| `src/navigation/types.ts` | Typed param lists for all navigators |
| `src/navigation/linking.ts` | Deep link config (scheme: `exporn`) |
| `src/navigation/navigationRef.ts` | Imperative navigation ref |
| `src/navigation/ThemedNavigationContainer.tsx` | NavigationContainer with theme |

Each tab owns a **nested native stack** so push/pop happens within the tab without affecting others.

---

## Theming and design tokens

### Theme system

- **`AppThemeProvider`** — light, dark, and system preference
- **`useAppTheme()`** — access current colors
- **`useThemedStyles()`** — create styles that react to theme changes
- **`AppColors`** — semantic color names (`text1`, `primary`, `background`, …)

Settings screen includes theme picker (light / dark / system).

### Design tokens workflow

Source files in `tokens/`:

| File | Purpose |
|---|---|
| `design.tokens` | Raw Figma export |
| `semantic-colors.json` | Maps semantic names to Figma color keys |

Regenerate after Figma updates:

```bash
npm run generate:tokens
```

Output lands in `src/designSystem/generated/` (spacing, font sizes, border radii, colors, etc.).

### UI conventions

- Use **`AppText`**, **`AppButton`**, **`AppList`**, **`AppTextInput`** — not raw RN primitives
- Use generated tokens for spacing and typography
- Use `useThemedStyles` for semantic colors

Details: [`tokens/README.md`](../tokens/README.md)

---

## Data layer

### API client

`src/api/client.ts` creates an Axios instance with:

- Base URL from `src/config/env.ts`
- Typed error handling via `src/api/types/errors.ts`
- Endpoint constants in `src/api/endpoints.ts`

### TanStack Query

| File | Purpose |
|---|---|
| `src/query/queryClient.ts` | Shared QueryClient with defaults |
| `src/query/QueryProvider.tsx` | React context provider |
| `src/query/queryKeys.ts` | Centralized query key factory |
| `src/query/hooks/useAppQuery.ts` | Typed query hook wrapper |
| `src/query/hooks/useAppMutation.ts` | Typed mutation hook wrapper |

### Example service

`src/services/postService.ts` demonstrates fetching data consumed by `PostsScreen` via query hooks.

---

## Internationalization

Powered by **i18next** with device locale detection.

| File | Purpose |
|---|---|
| `src/third-party/i18n/i18n.ts` | i18next initialization |
| `src/third-party/i18n/locales/en/translation.json` | English strings |
| `src/third-party/i18n/locales/es/translation.json` | Spanish strings |
| `src/third-party/i18n/changeAppLanguage.ts` | Runtime language switch |
| `src/third-party/i18n/supportedLanguages.ts` | Available locales |

Imported at the top of `App.tsx` before any component renders.

Settings screen includes a language picker bottom sheet.

Usage in screens:

```tsx
const { t } = useTranslation();
return <AppText>{t("tabs.home")}</AppText>;
```

---

## Shared utilities

### Loading

Global loading overlay and inline states via `LoadingProvider`:

- `LoadingOverlay` — full-screen blocking loader
- `InlineLoading` — inline spinner
- `SkeletonView` — placeholder skeletons

### Empty and error states

- `EmptyStateView` — no data placeholder
- `ErrorStateView` — recoverable error UI

### Error boundary

`AppRootErrorBoundary` catches render errors at the app root with a fallback screen.

### Connectivity

`ConnectivityProvider` wraps NetInfo for online/offline awareness.

### Local storage

MMKV-backed storage in `src/third-party/localstorage/` with typed keys in `LocalStorageKeys.ts`.

---

## Quality gates

Run locally before pushing:

```bash
npm run lint          # ESLint (expo lint)
npm run format:check  # Prettier
npm run typecheck     # tsc --noEmit
npm run test          # Jest
npm run test:ci       # Jest with coverage (CI threshold)
```

Fix formatting:

```bash
npm run format
```

### Test setup

- Config: `jest.config.js`, `jest.setup.js`
- Example test: `__tests__/App.test.tsx`
- Coverage output: `coverage/` (git-ignored in practice via CI artifacts)

---

## CI integration notes

This template is designed to integrate with the shared **`rn.yml`** workflow from `template-pipeline-react-native`. Key requirements:

1. **Run prebuild** with the correct `APP_ENV`, then commit `android/` and `ios/`
2. **Keep root `Gemfile` / `Gemfile.lock`** for Bundler (CocoaPods in CI)
3. **Apply Fastlane** from the DevOps kit after prebuild
4. **Bootstrap workflow IDs:** `python3 .github/scripts/bootstrap_rn_workflow_ids.py` after native project generation
5. **Configure signing secrets** per the pipeline repo's checklist

JS gates in CI match local scripts: `lint`, `format:check`, `typecheck`, `test:ci`.

Full CI notes: [`README.md`](../README.md#ci-github-actions)

---

## Quick reference — npm scripts

| Script | Action |
|---|---|
| `npm start` | Metro (development) |
| `npm run ios:dev` | Prebuild + run iOS dev |
| `npm run android:qa` | Prebuild + run Android staging |
| `npm run generate:tokens` | Regenerate design tokens from Figma |
| `npm run lint` | ESLint |
| `npm run typecheck` | TypeScript check |
| `npm run test:ci` | Tests with coverage |
