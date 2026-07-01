# Share links → App Store / Play Store

## Goal
When someone opens a share link (`/:referenceCode`) on a phone **without the app**,
send them straight to the right store. On desktop they see the card normally, with
store buttons in the card header. Also fix the placeholder store links on the
landing page, and add a header (logo left, store buttons right) to all share cards.

> Note: with Universal Links (iOS) and App Links (Android) configured, users who
> already have the app never hit the web page — the OS opens the app directly. So
> the redirect only ever fires for people without the app, which is exactly who we
> want sending to the store.

## Store URLs (real, from user)
- iOS: `https://apps.apple.com/us/app/nanza/id6757965694`
- Android: `https://play.google.com/store/apps/details?id=com.oakplatforms.nanza&hl=en_US`

## Changes

### 1. Centralize store URLs — `src/constants.ts` (additive)
```ts
export const APP_STORE_URL = 'https://apps.apple.com/us/app/nanza/id6757965694'
export const PLAY_STORE_URL =
  'https://play.google.com/store/apps/details?id=com.oakplatforms.nanza&hl=en_US'
```
Confidence: 0.95 — single source of truth, used by landing + share.

### 2. Device detection helper — `src/helpers/getMobilePlatform.ts` (new)
Returns `'ios' | 'android' | null` from `navigator.userAgent`. iPadOS 13+ reports as
Mac, so also treat `navigator.maxTouchPoints > 1` on Mac as iOS.
Confidence: 0.9 — standard UA sniff; acceptable for a redirect-to-store nudge.

### 3. Mobile auto-redirect — `src/app/ShareDetail/index.tsx`
At the top of `ShareDetail` (after the code is validated as a real reference code),
detect platform; if iOS/Android, `window.location.replace(storeUrl)` and render
nothing. Desktop falls through to the normal card. Use `replace` so Back doesn't
trap them on the share page.
Confidence: 0.9 — gate on a valid reference code so we never redirect on garbage URLs.

### 4. Share card header — `src/app/ShareDetail/ShareCardHeader.tsx` (new)
Replaces the centered `ShareFooterLogo` beneath the card. Sits **inside** the card
container (top of `ShareDetailShell`'s inner div): Nanza logo top-left (links to `/`),
two store buttons on the far right. Reuses the same store-button markup as the landing.
Render it once in `ShareDetailShell` so every variant (Listing/Bid/Product/List/Bulk)
gets it for free, and drop the per-variant `<ShareFooterLogo />` calls.
Confidence: 0.85 — putting it in the shell avoids touching all five layouts; alt
considered: add to each layout (rejected — repetition, P2 DRY).

### 5. Fix landing store links (real URLs)
- `src/landing/index.tsx` — App Store + Google Play `href`s
- `src/landing/components/LandingLayout.tsx` — App Store + Google Play `href`s
Confidence: 0.97 — swap placeholder hrefs for the constants.

## Files touched
- `src/constants.ts` (additive)
- `src/helpers/getMobilePlatform.ts` (new)
- `src/app/ShareDetail/index.tsx`
- `src/app/ShareDetail/ShareCardHeader.tsx` (new)
- `src/app/ShareDetail/ShareDetailShell.tsx` (render header)
- `src/landing/index.tsx`
- `src/landing/components/LandingLayout.tsx`
- remove uses of `ShareFooterLogo` (keep file or delete)

## Open question
On mobile, the redirect means the card never shows. The header store-buttons are
therefore desktop-only in practice. That's fine and intended.
