# Solution Design — "Update Available" Bottom Sheet

**Status:** BUILT (2026-06-30). Backend + mobile implemented, both repos
tsc-clean, not committed. Remaining: set the `LATEST_APP_VERSION_DEV` /
`LATEST_APP_VERSION_PROD` GitHub variables, fill the Apple App Store ID in
`useUpdateSheet.tsx`, redeploy the API.
**Author:** Claude (with Skylar)
**Date:** 2026-06-30

---

## Goal

On app load, if the installed app version is older than the latest released
version, show a bottom sheet:

> **A new version of Nanza is available**
> New features and improvements are ready. Update to get the latest.
> **[ Update ]**

- **Update button** → opens the App Store (iOS) or Play Store (Android) listing.
- **Dismissable** — backdrop tap / drag-down closes it (reusing the existing
  `BottomSheet` dismiss behaviour).
- **Once per app load** — shows at most once per cold start. If dismissed, it
  does not reappear until the app is reloaded (fresh launch).

This mirrors the recently-shipped **usage-caps bottom sheet** (`useCapSheet` +
`ModalStack` + `ConfirmSheetContent`). We reuse that whole stack; this is
additive.

---

## How it works (the mechanism)

This is **not** an error path. On cold start the app:

1. Reads its **own installed version** via `react-native-device-info`'s
   `getVersion()` (already a dependency, already used in `AnalyticsBridge`).
2. Reads the **latest released version** from the backend (a single string,
   e.g. `"0.3.0"`).
3. Compares them with semver. If `installed < latest`, push the bottom sheet.
   If equal/newer, do nothing — the user never sees it.

**Future-proofing note (confirmed with Skylar):** the comparison logic ships
*inside* the app, so only builds from this release forward can be prompted.
Users on older builds without the check won't see it — by design.

---

## Where the "latest version" comes from

**DECISION (locked): a dedicated one-time `GET /app-version` endpoint, called
once on app load.** Cleaner separation than piggybacking on a domain response,
and it fires for **everyone** — including guests on the splash/onboarding flow,
not just signed-in users.

### Why a dedicated endpoint (vs. piggybacking)

There is no single existing call that fires for **every** user on **every** cold
start today:

- `App.tsx` startup `useEffect` → only analytics (`initAnalytics`,
  `trackInstallOnce`, `trackAppOpen`). No nanza-api call.
- `SessionContext` → `fetchAuthSession()` is **Cognito**, not our API. The first
  nanza-api call (`userService.get`) is **gated behind sign-in**
  (`SessionContext.tsx:81-85`), so guests would never get a version field there.

A dedicated `GET /app-version` sidesteps both: one cheap, cacheable call on
launch that reaches guests and signed-in users alike, and keeps app-version
metadata out of domain payloads.

### Chosen path

`nanza-api` adds `GET /app-version` returning `{ latest: string, minimum?: string }`.
Source of truth is a single configured value (env var / config row) bumped each
release. The mobile app calls it once on load via a small read-hook and compares
`latest` against `getVersion()`.

> Alternative rejected: attaching `latestAppVersion` to the existing user GET
> response (no extra request). Rejected because it only reaches signed-in users
> and couples app-version metadata to a domain payload. The dedicated endpoint
> is one cheap cached call and keeps the concern isolated.

---

## Components & files

All new files are additive. No navigation, no shared-infra, no API `fetchData`
changes.

### 1. Version compare util — `src/utils/version.ts` (new)
A tiny semver-ish comparator. No new dependency.

```ts
// Returns true when `current` is strictly older than `latest`.
// Handles "0.2.6" vs "0.3.0" by numeric segment compare; tolerates
// missing segments and non-numeric suffixes by ignoring them.
export const isOutdated = (current: string, latest?: string): boolean => {
  if (!latest) return false
  const parse = (v: string) =>
    v.split('.').map(s => parseInt(s, 10)).map(n => (Number.isFinite(n) ? n : 0))
  const a = parse(current)
  const b = parse(latest)
  const len = Math.max(a.length, b.length)
  for (let i = 0; i < len; i++) {
    const x = a[i] ?? 0
    const y = b[i] ?? 0
    if (x !== y) return x < y
  }
  return false
}
```
**Confidence: 0.9** — small, pure, unit-testable. Rejected alternative: pulling
in the `semver` npm package — overkill for a 3-segment compare and adds a dep.

### 2. Store-link helper — inside the hook (below)
```ts
import { Linking, Platform } from 'react-native'

// TODO(values): fill in once the listings are live.
const APP_STORE_URL = 'https://apps.apple.com/app/id<APPLE_APP_ID>'
const PLAY_STORE_URL =
  'market://details?id=com.oakplatforms.nanza' // falls back to web below

const openStore = () => {
  const url = Platform.OS === 'ios' ? APP_STORE_URL : PLAY_STORE_URL
  Linking.openURL(url).catch(() => {
    // Play Store web fallback if the market:// intent has no handler
    if (Platform.OS === 'android') {
      Linking.openURL(
        'https://play.google.com/store/apps/details?id=com.oakplatforms.nanza',
      )
    }
  })
}
```
Android package is confirmed: `com.oakplatforms.nanza`. **The Apple App Store
numeric ID is not in the repo** — we need it from App Store Connect. (Marked as
a value to fill, not a blocker for the design.)

### 3. `useUpdateSheet` hook — `src/hooks/useUpdateSheet.tsx` (new)
Direct sibling of `useCapSheet`. Reuses `useModalStack` + `ConfirmSheetContent`.

```ts
export const useUpdateSheet = () => {
  const { pushModal, popModal } = useModalStack()

  const showUpdateSheet = useCallback(() => {
    const modalId = 'update-available'
    pushModal({
      id: modalId,            // pushModal de-dupes by id → can't double-show
      showHeader: false,
      scrollable: false,
      content: (
        <ConfirmSheetContent
          title="A new version of Nanza is available"
          subtitle="New features and improvements are ready. Update to get the latest."
          confirmLabel="Update"
          onConfirm={async () => {
            openStore()
            popModal(modalId)
          }}
        />
      ),
    })
  }, [pushModal, popModal])

  return { showUpdateSheet }
}
```
**Confidence: 0.92** — structurally identical to `useCapSheet`; reuses the same
content component and dismiss model (backdrop/drag close it for free).

### 4. Trigger mount-point — `src/components/global/UpdateGate.tsx` (new)
`useModalStack` only works **inside** `AppNavigator` (where `ModalStackProvider`
lives). So the trigger can't sit in `App.tsx` directly. We add a tiny headless
component mounted inside the navigator tree (next to where `ModalStackRenderer`
already is) that runs the check **once per app load** via a ref guard.

```tsx
const UpdateGate = () => {
  const { showUpdateSheet } = useUpdateSheet()
  const { data } = useAppVersion()           // one-time GET /app-version
  const checked = useRef(false)

  useEffect(() => {
    if (checked.current) return
    const latest = data?.latest
    if (!latest) return                // wait until the request resolves
    checked.current = true             // once-per-load guard
    if (isOutdated(getVersion(), latest)) showUpdateSheet()
  }, [data, showUpdateSheet])

  return null
}
```
- **Once per load** = the `checked` ref. New cold start = new component instance
  = ref resets = checks again. Exactly the requested behaviour.
- Mounted once, near `ModalStackRenderer` in `AppNavigator.tsx`.

**Confidence: 0.8** — headless trigger is a clean pattern, but it touches
`AppNavigator.tsx` to mount one component (a P0-sensitive file). It is **not** a
navigation/registration change — just adding `<UpdateGate />` next to the
existing `<ModalStackRenderer />`. Flagged for approval. Rejected alternative:
firing from `App.tsx` startup effect — can't, `useModalStack` isn't in scope
there.

### 5. Service + read-hook — `src/services/api/AppVersion.ts` + `src/hooks/useAppVersion.ts` (new)
Standard service + read-hook following the `InboxCount` pattern.

```ts
// src/services/api/AppVersion.ts
export const appVersionService = {
  async get() {
    return fetchData({ url: '/app-version' })
  },
}
```
```ts
// src/hooks/useAppVersion.ts — one-time fetch on load, no polling
export const useAppVersion = () =>
  useQuery<AppVersionResponse>({
    queryKey: ['appVersion'],
    queryFn: () => appVersionService.get(),
    staleTime: Infinity,
    refetchOnWindowFocus: false,
  })
```
`AppVersionResponse` here is `{ latest: string; minimum?: string }`. This is a
small **response shape for an infra endpoint**, not a backend domain DTO — it
does not mirror a `@oakplatforms/types` model, so a local type is appropriate
(same as `InboxCountResponse` in `useInboxCount.ts`). If the team prefers, the
shape can be added to `@oakplatforms/types` instead.

### 6. Backend — `GET /app-version` (nanza-api)
New lightweight route returning `{ latest, minimum? }` from a single configured
value bumped each release. No auth required (so guests get it too).

---

## Styling

**No new styles needed.** `ConfirmSheetContent` already renders title +
subtitle + full-width button using `getModalStyles(theme)` from
`src/styles/components/modals.ts`. Every token is theme-based. **Zero P0 style
risk** — we render the existing component with different copy.

---

## What this touches

| File | New/Edit | Notes |
|---|---|---|
| `src/utils/version.ts` | new | pure compare util |
| `src/services/api/AppVersion.ts` | new | `GET /app-version` service |
| `src/hooks/useAppVersion.ts` | new | one-time read-hook |
| `src/hooks/useUpdateSheet.tsx` | new | sibling of `useCapSheet` |
| `src/components/global/UpdateGate.tsx` | new | headless once-per-load trigger |
| `src/navigation/AppNavigator.tsx` | **edit** | mount `<UpdateGate />` only (P0-sensitive — flagged) |
| nanza-api | **edit** | new `GET /app-version` route → `{ latest, minimum? }` |

Frontend footprint: **5 new files + 1 one-line navigator edit**. At the "≤5
files" guideline (the navigator edit is the only existing-file touch); the
navigator touch is the only escalation-worthy item.

---

## P0 / P1 compliance check

- ✅ No inline styles / `StyleSheet.create` — reuses `ConfirmSheetContent`.
- ✅ No hardcoded colors / sizes — all via existing themed component.
- ✅ Dollar-sign rule — N/A (no prices).
- ⚠️ **Navigation file edit** — mounting `<UpdateGate />` in `AppNavigator.tsx`.
  Not a route/registration change, but the file is P0-protected → **escalated**
  below for explicit sign-off.
- ✅ No `any`, no custom DTOs — version values are primitives; the
  `{ latest, minimum? }` response is an infra shape (like `InboxCountResponse`),
  not a domain DTO. If preferred it can live in `@oakplatforms/types` instead.
- ✅ API change only in service file (`AppVersion.ts`) / backend — `fetchData`
  untouched.

---

## Escalation — navigation file touch

```
Proposed change:
- Files: src/navigation/AppNavigator.tsx (add one line: <UpdateGate />)
- Impact: mounts a headless trigger next to the existing <ModalStackRenderer />

Cause for concern:
- AppNavigator.tsx is P0-protected.

Why this is the best option:
- useModalStack only resolves inside ModalStackProvider, which lives in
  AppNavigator. The trigger must mount inside that tree. Confidence: 0.8

Alternatives considered:
1. Fire from App.tsx startup effect - Confidence: 0.3 - Rejected: useModalStack
   out of scope there; would need a second modal mechanism.
2. Trigger lazily from a screen (e.g. Home) - Confidence: 0.5 - Rejected: misses
   "on app load" for deep-link / non-Home cold starts.

Risk mitigation:
- Additive single line; no route, param, or registration-order change.
- Headless component returns null; no layout/visual impact.
```
