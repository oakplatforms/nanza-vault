# Architecture

How nanza-mobile is wired together end to end: app entry, navigation, the data/state layer,
the styling system, the services/api layer, and how types flow in from `@oakplatforms/types`.
This is the structural companion to [[REFERENCE|Frontend Reference Guide]] (which holds the
style rules, token tables, and pattern catalog) and [[INDEX|the project map]]. Every claim
below is grounded in a real file path under `src/`.

---

## App entry & provider tree

`App.tsx` (repo root) is the entry. It builds a single `QueryClient` and wraps the app in a
nested provider tree. Reading `App.tsx` top to bottom, the order is:

```
QueryClientProvider
  AuthProvider            (src/contexts/AuthContext.tsx)
    SessionProvider       (src/contexts/SessionContext.tsx)
      AnalyticsBridge
      CartProvider        (src/contexts/CartContext.tsx)
        SavedItemsProvider(src/contexts/SavedItemsContext.tsx)
          CollectionSelectionProvider (src/contexts/CollectionSelectionContext.tsx)
            ThemeProvider (src/styles/utils/ThemeProvider.tsx)
              AppRoot
```

`AppRoot` adds three more providers (`ShareGuardProvider`, `ListingQuantityProvider`) around
`AppNavigator`, and renders `SplashAnimation` on top until it signals complete.

Two app-lifecycle concerns live in `App.tsx`:

- **React Query focus refetch** — `focusManager.setEventListener` bridges React Native
  `AppState` to React Query so queries can refetch when the app returns to the foreground
  (guarded to non-web).
- **Analytics lifecycle** — `initAnalytics`, `trackInstallOnce`, and `trackAppOpen` fire from
  `src/services/analytics/`; an `AppState` listener emits exactly one `app_open` per genuine
  foreground (see the comment in `App.tsx` about the iOS background→inactive→active dance).

The `QueryClient` defaults (in `App.tsx`) set `staleTime` 5 min, `gcTime` 10 min, `retry: 0`,
`refetchOnWindowFocus: false`, `refetchOnMount: true`. Individual hooks override `staleTime`
via the `STALE_TIME` constants (see [[REFERENCE|Frontend Reference Guide]]).

---

## Navigation (read-only)

Three files, all under `src/navigation/` and treated as **read-only** (never edit screen
registration, params, or order — escalate instead):

- **`AppNavigator.tsx`** — the root `NavigationContainer` + a `createStackNavigator`. It
  registers `BottomTabNavigator` as the base route and then dozens of pushed screens
  (`EntityScreen`, `BuyScreen`, the `Create*`/`Edit*` screens, the scan flow
  `ScanCameraScreen`/`ScanReviewScreen`/`ScanItemsScreen`, the trade flow, order/offer
  detail, wallet, legal, etc.). It also wires cross-cutting pieces: `AlertProvider`,
  `ModalStackProvider` + `ModalStackRenderer`, and `UpdateGate` (the update-available gate).
- **`BottomTabNavigator.tsx`** — the `createBottomTabNavigator` with the primary tabs (Home,
  Messages, Trade/Lightning, Groups, Search, Profile). Several tabs wrap their screen in a
  small per-tab `createStackNavigator` (e.g. `HomeStack` pushes `UserProfileScreen`). It also
  consumes app state directly: `useShoppingCart`, `useSession`, `useInboxCount` (envelope
  badge), and `useFetchBrands`, and renders `AuthModal` and `CartPreview`.
- **`ProfileNavigator.tsx`** — the stack for the Profile tab.

Deep linking is configured in `AppNavigator.tsx`: it imports `SHARE_BASE_URL` from `@env` and
`parseShareReferenceTarget` from `src/utils/referenceCode` to turn incoming
`nanza.app/<refCode>` links into navigation targets. This is the mobile end of the
self-hosted shareability system (`S`/`B`/`C`/`G`/`U` reference codes).

Screen transitions use `CardStyleInterpolators` from `@react-navigation/stack`.

---

## State & data layer

State is split by lifetime: **server state** lives in React Query; **cross-cutting client
state** lives in the contexts below.

### React Query hooks

The read pattern is a `useFetchX` hook per screen, colocated in that screen's `data/` folder
(e.g. `src/screens/search/data/fetchSearch.ts`,
`src/screens/home/data/fetchBrands.ts`). Each hook calls a service module and returns a
feature-named object (`{ brands, refetch, isLoading }` rather than the raw query result).
Paginated data uses `useInfiniteQuery`; `enabled` guards on required params; `queryKey`
includes the key plus identifying params; `staleTime` comes from `STALE_TIME` in
`src/constants.ts`. Hooks are read-only by default — mutations are added only when a feature
needs them. The exact conventions and a worked example are in
[[REFERENCE|Frontend Reference Guide]].

Some cross-screen hooks live in `src/hooks/` instead of a screen `data/` folder because they
aren't owned by one screen — e.g. `useFetchByReference.ts` (resolves any share reference code
to its DTO), `useInboxCount.ts`, `useAppVersion.ts`, and the sheet hooks
`useUpdateSheet.tsx` / `useCapSheet.tsx`.

### Contexts

Under `src/contexts/`, each is a `createContext` + provider + `useX` hook:

- **`SessionContext.tsx`** — the current user and derived access flags. Its `TSessionContext`
  exposes `currentUser: UserDto`, `isSignedIn`, `refreshCurrentUser`, access booleans
  (`hasSellerAccess`, `hasCustomerAccess`, `hasSellerPaymentMethod`,
  `hasCustomerPaymentMethod`, `isRegistered`), and the `selectedBrandId` feed filter. It
  reads identity from `src/services/auth/cognito` and hydrates the user via `userService`.
- **`AuthContext.tsx`** — auth/sign-in input state and flows (Cognito).
- **`CartContext.tsx`** — the shopping cart (`useShoppingCart`), backed by
  `src/contexts/data/fetchCartOrders.ts`.
- **`SavedItemsContext.tsx`** — saved/favorited items.
- **`CollectionSelectionContext.tsx`** — the currently active collection for the collect/scan
  flows.
- **`ListingQuantityContext.tsx`** — per-listing quantity selection (`useListingQuantity`).
- **`ModalStackContext.tsx`** — a cascade of stacked modals rendered by
  `src/components/modals/ModalStack`, mounted from `AppNavigator`.
- **`ShareGuardContext.tsx`** — guards the share deep-link entry so a cold-start share link
  routes correctly; exposes a module-level `setShareGuardActive`.

Contexts, `styles/utils/*`, and `fetchData` are shared infrastructure and off-limits without
escalation (see the project `CLAUDE.md`).

---

## Styling system

The chain is **`dark.json` tokens → ThemeProvider → `ThemeTokens` → style-bucket getters →
component classes**. No component writes inline style objects (the P0 rule set lives in
`CLAUDE.md` / [[REFERENCE|Frontend Reference Guide]]).

### Tokens → theme

`src/styles/tokens/dark.json` is the single source of design values. `ThemeProvider.tsx`
resolves it once at module load:

```
resolveThemeColors(dark.json)  →  parseNumericStrings(...)  →  fallbackTheme
```

`resolveThemeColors` (`src/styles/utils/resolveThemeColors.ts`) flattens color scales, and
`parseNumericStrings` (`src/styles/utils/parseNumericStrings.ts`) turns stringified numbers
into numbers, producing a `ThemeTokens` object. `ThemeTokens` is typed in
`src/styles/utils/types.ts` (`colors`, `spacing`, `radius`, `icons`, `fonts`, …). The
provider currently serves this single resolved dark theme and exposes `{ theme, themeMode,
setThemeMode }` via the `useTheme()` hook.

### Theme → styles

Style-bucket files live in `src/styles/components/*.ts` — one file per UI area (`layout.ts`,
`typography.ts`, `buttons.ts`, `cards.ts`, `inputs.ts`, `badges.ts`, `entity.ts`,
`panels.ts`, `headers.ts`, `tabs.ts`, `modals.ts`, `menus.ts`, `share.ts`, `scan.ts`, …).
Each exports a `getXStyles(theme: ThemeTokens)` that returns a typed map of named classes
built from theme tokens (never hardcoded values). `src/styles/components/index.ts` composes a
few of these into `getThemeStyles(theme)` for convenience.

### How a component consumes styles

```typescript
import { useTheme } from '../../../styles/utils/ThemeProvider'
import { getButtonStyles } from '../../../styles/components/buttons'

const { theme } = useTheme()
const buttonStyles = getButtonStyles(theme)
// ...
<View style={buttonStyles.buttonCTAContent}>...</View>
```

Truly dynamic runtime values (Animated transforms, window-height-derived sizes) may stay
inline — see `getPromoAndroidStyles` in `src/styles/components/layout.ts` for an example that
computes sizes off live window height and is intentionally not static. The full before/after
catalog (16 patterns) is in [[REFERENCE|Frontend Reference Guide]].

Fonts: EuclidCircularB (primary), Geist (secondary), Figtree (prices/dollar sign, rendered
+1 weight above the numeric portion — the `PriceText` rule). Font families and the price/
dollar-sign convention are tabled in [[REFERENCE|Frontend Reference Guide]].

---

## Services / API layer

Every backend call funnels through **`fetchData`** in `src/services/api/index.ts`. Its
responsibilities:

- **Auth token resolution** — `getJwtToken()` tries a Cognito access token via
  `fetchAuthSession` (`src/services/auth/cognito`); on failure it falls back to a cached
  **guest token** from `GET /user/guest-token` (cached ~10 min). The token is attached as
  `Authorization: Bearer …`.
- **Base URL** — prepends `API_BASE_URL` from `@env` (per-environment `.env` files in the repo).
- **Body handling** — JSON by default, but passes `FormData` through untouched (for image
  uploads) and omits the `Content-Type` header in that case.
- **Error semantics** — a `429` throws `RateLimitError`; a non-OK response parses the error
  body and throws. A `403` with `errorCode: 'CAP_REACHED'` throws a typed **`CapError`**
  carrying the `feature` (`'listing' | 'bid' | 'message' | 'collection' | 'scan'`), which the
  client catches to show the freemium cap bottom sheet (`useCapSheet`) instead of a generic
  error. Other failures throw a named `APIError`.

On top of `fetchData` sit **thin per-resource service modules** — one file per backend
resource in `src/services/api/` (`Entity.ts`, `Listing.ts`, `Bid.ts`, `Order.ts`,
`Profile.ts`, `Group.ts`, `BulkListing.ts`, `Feed.ts`, `SellFeed`-style feeds,
`Reference.ts`, `AppVersion.ts`, `InboxCount.ts`, and ~40 more). Each exports a service
object whose methods build a URL path and delegate to `fetchData`, e.g.
`fetchData({ url: '/entity/${id}${params}' })`. Endpoint changes are made **only** in these
service files, preserving the `fetchData` call and the `/resource/...` path convention.

`Reference.ts` is a good example of a service that does real work before the call: its
`buildIncludes` assembles per-type `include` query params (mirroring the web app's
share-detail fetch) so that share cards for each reference type (`S`, `B`, `C`, `G`, `U`)
receive the correct populated relations.

Alongside `api/`, `src/services/` also has `analytics/` (event tracking + lifecycle) and
`auth/` (Cognito session helpers).

---

## Type flow from `@oakplatforms/types`

Domain types are **not** defined locally. `src/types/index.ts` imports from the published
`@oakplatforms/types` package and re-exports the DTOs the app uses (`UserDto`, `EntityDto`,
`ListingDto`, `BidDto`, `OrderDto`, `ProfileDto`, `BulkListingDto`, `ListDto`,
`PaginatedResponse`, …), plus the raw OpenAPI `components` map. Components and hooks type
against these DTOs; if a DTO looks missing the package is out of date and gets bumped rather
than a stand-in being invented (a hard project rule).

Two kinds of local types are legitimate and stay in-repo:

- **Route param types** — `src/types/navigation.ts` (`RootStackParamList`, etc.) and
  component prop types are UI concerns, not domain DTOs.
- **Response/route envelope shapes** — a few response wrappers that aren't domain models live
  next to their consumer. For example `useFetchByReference.ts` defines `ReferenceRecordType`,
  `ProfileShareData`, and `ProfileShareSummary` locally, but their *inner* records are real
  `@oakplatforms/types` DTOs — the local type is only the envelope/aggregate around them
  (analogous to a `commentCount`).

---

## See also

- [[INDEX|Project map]] — structure + plans by theme
- [[REFERENCE|Frontend Reference Guide]] — style rules, token tables, pattern catalog
- [[../_shared/INDEX|Shared brain]] — cross-project architecture & catalog
