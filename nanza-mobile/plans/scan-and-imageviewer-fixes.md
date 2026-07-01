# Scan + Image Viewer Fixes

Five fixes across three areas. Two are **frontend-only** (mobile `src/`), one needs a **backend API change** (nanza-api), and the image-viewer work is **frontend-only** but touches a shared component.

---

## 1. Scan results carousel won't scroll left/right on Android (FE-only)

**Where:** `src/screens/trade/ScanCameraScreen/index.tsx:445-460` — a bare horizontal `<ScrollView>`.

**Root cause (hypothesis, Confidence: 0.78):** The `ScanCamera` screen is registered with `gestureDirection: 'horizontal'` and `gestureEnabled: true` in `AppNavigator.tsx`. On Android, react-native-gesture-handler's screen-level horizontal swipe-back PanGestureHandler competes with the child ScrollView and wins, so the carousel never claims the horizontal pan. iOS delegates to the child more gracefully.

**Fix options (in order of preference):**
- **A (Confidence: 0.8):** Add `nestedScrollEnabled` + `directionalLockEnabled` to the ScrollView, and wrap it so its horizontal pan is recognized. Cheapest, no nav change. Rejected if it doesn't take.
- **B (Confidence: 0.75):** Swap the bare `ScrollView` for `ScrollView` from `react-native-gesture-handler` (already a dependency), which coordinates with the screen gesture handler properly. **Recommended** — purpose-built for exactly this nested-gesture conflict.
- **C:** Disable the screen's horizontal swipe-back gesture while the variant picker is open. Rejected — touches navigation (P0 #7) and changes screen behavior.

> ⚠️ **Verification caveat:** I can't reproduce on a physical Android device from here. The fix is a strong hypothesis; you'll need to confirm on-device.

---

## 2. Scan dedup: listings and collections should be independent (BACKEND change — escalation)

**Current behavior (the bug):** `nanza-api/src/routers/scan.ts:215-246` (`POST /scan/cards`) excludes any entity the user has an ACTIVE listing for **OR** any existing ScanItem for, regardless of scan mode. So a card you've listed is hidden when scanning into a collection, and vice versa.

**Desired behavior:**
- Scanning for a **LISTING** → only hide cards you already have an ACTIVE **listing** for (and LISTING-type scan items).
- Scanning into a **COLLECTION** → only hide cards already in that collection / COLLECTION-type scan items. Do **not** hide for listings.

**Why this is a backend change:** The `/scan/cards` endpoint doesn't currently receive the scan mode, so it can't tell which dedup to apply. The mobile client already knows the mode (`ScanCameraScreen` line 141-143: `scanMode`/`scanItemType`).

**Plan:**
- **API (nanza-api):** add a `mode`/`scanType` param to `POST /scan/cards`; branch the exclusion query — `LISTING` mode excludes ACTIVE listings + LISTING scan items; `COLLECTION` mode excludes COLLECTION scan items (optionally the target list's contents). Confidence: 0.85.
- **Mobile (`src/services/api/*` + `ScanCameraScreen`):** pass `scanItemType` (already computed) through the scan call. Surgical, additive. Confidence: 0.9.

> This crosses into nanza-api and is **outside the mobile `src/` scope** — flagging per project rules. Need your go-ahead to edit the API repo. (Collection dedup against a *specific* list vs. all collections is a product decision — see open question below.)

---

## 3. Image zoom viewer — three sub-fixes (FE-only, shared component)

**Component:** `src/components/global/ImageViewer/index.tsx` + `src/styles/components/imageViewer.ts`. Used by FeedListingCard, FeedBulkCard, FeedBidCard, OrderItems. This is a **shared component** — changes here ripple to all call sites, so the carousel addition must stay backwards-compatible (single-image callers keep working).

### 3a. Consistent image height
- **Cause:** `resizeMode="contain"` with `width/height: '100%'` → displayed height varies by each image's aspect ratio.
- **Fix (Confidence: 0.72):** Constrain the image to a fixed aspect-ratio frame (e.g. a card ratio) and decide stretch vs. crop. You said uniform-even-if-cropped is acceptable. Recommended: a fixed-ratio centered frame with `resizeMode="cover"` so every image fills the same box. **Open question below** on exact ratio.

### 3b. Android backdrop doesn't reach top/bottom on tall phones
- **Cause:** `<Modal>` lacks `statusBarTranslucent` and the backdrop is `flex: 1` without forcing full-window coverage.
- **Fix (Confidence: 0.85):** Add `statusBarTranslucent` (and on Android, `navigationBarTranslucent` if supported) to the Modal so the backdrop extends edge-to-edge. Keep close button on `insets.top`. Move colors per P0 (already theme-based). Recommended.

### 3c. Lots → swipeable carousel in the viewer
- **Cause:** `ImageViewer` takes a single `source`. Lots have an array of images (FeedBulkCard already builds a `slides` array at lines 113-119) but only pass the tapped URI.
- **Fix (Confidence: 0.7):** Extend `ImageViewer` to optionally accept an array of sources + an initial index, render a horizontal paged carousel (each page keeps pinch/pan/double-tap), and a page indicator. Single-image callers unchanged (pass one source as today). FeedBulkCard passes its `slides` array + tapped index. Bids/single listings stay single. Recommended — additive, backwards-compatible.

---

## Files touched (estimate)

**Mobile (FE):**
1. `src/screens/trade/ScanCameraScreen/index.tsx` — #1 ScrollView swap, #2 pass scan mode
2. `src/services/api/*` (scan endpoint) — #2 pass mode param
3. `src/components/global/ImageViewer/index.tsx` — #3a, #3b, #3c
4. `src/styles/components/imageViewer.ts` — #3a, #3b styles
5. `src/components/feed/FeedBulkCard/index.tsx` — #3c pass slides array + index

**Backend (escalation, separate repo):**
6. `nanza-api/src/routers/scan.ts` — #2 mode-aware dedup

That's 5 mobile files + 1 API file. The mobile count is within the 5-file guideline; the API file is the escalation.

---

## Open questions (need answers before coding)

1. **Collection dedup scope (#2):** When scanning into a collection, hide cards already in *that specific collection*, or in *any* of your collections? (`ScanCamera` is launched with a `listId` for locked collections.)
2. **Viewer image ratio (#3a):** What aspect ratio should the uniform frame use — standard trading-card portrait (~5:7), or square? And confirm crop-to-fill (`cover`) over letterbox.
3. **API repo go-ahead (#2):** OK to make the dedup change in nanza-api, or mobile-only for now?
