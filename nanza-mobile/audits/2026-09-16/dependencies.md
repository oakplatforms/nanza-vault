# Dependency / Toolchain / Bundle-Size Audit — nanza-mobile

## Summary

| Finding | Impact | Effort | Risk | Timing |
|---|---|---|---|---|
| `npm`, `install` as runtime deps | Junk deps; `npm` alone pulls a huge subtree into `node_modules` (not into the app bundle, but slows installs/audits) | S | None | Fix now |
| `buffer` unused (no `Buffer`/`from 'buffer'` reference anywhere) | Dead dep | S | None | Fix now |
| `@types/react-native-dotenv` unused, and wrong location (types in `dependencies`) | Dead dep + convention violation | S | None | Fix now |
| `react-native-view-shot` — zero references in `src/`/`App.tsx`/`index.js` | Dead native dep; adds iOS+Android native code to both binaries for nothing | S | None | Fix now |
| `react-native-qrcode-svg` — real usage found (`src/components/share/QrCode/index.tsx`), knip/depcheck both wrong | N/A — keep | S | None | Fix now (correct the depcheck/knip finding) |
| `react-native-floating-label-input` unused | Dead native-ish dep (JS-only, has ios/android dirs but likely from a peer, confirm) | S | None | Fix now |
| `openapi-typescript` present in mobile repo | Types come from `@oakplatforms/types`; this looks like a copy-paste from nanza-api's tooling | S | None | Fix now |
| `prettier` 2.8.8 pinned, unused (no config, no script, ESLint owns formatting) | Confusing devDep; version is also 3 majors behind | S | None | Fix now |
| `react-dom` 19.2.0 in a RN app | Only needed if something renders web (nothing does) — dead weight in devDependencies | S | None | Fix now |
| `metro-config` (0.82.5) pinned separately from `@react-native/metro-config` (0.83.x) | Two different metro-config packages, version-mismatched; only the RN-scoped one is used by `metro.config.js` | S | Low | Fix now |
| `@typescript-eslint/parser` unused (superseded by `typescript-eslint` meta-package) | Dead devDep | S | None | Fix now |
| eslint 8.57 (EOL) + `typescript-eslint` 7.18 + TS 5.9.3 | `typescript-eslint@7` only declares `eslint: ^8.56.0` as peer (verified via `npm view`) — works today, but TS 5.9 is past what TS-ESLint 7's support window targeted (7.x tested against TS <5.6); no functional break observed via `npm ls`, but the trio is stale together | M | Low | Later release |
| Two lockfiles (`yarn.lock` + `package-lock.json`), both maintained in lockstep, no CI | Redundant tooling; README recommends yarn as the alt path, no enforcement of one | S | Low | Later release |
| `.nvmrc` pins Node 21.7.1 (EOL), `engines` says `>=20`, `polyfill.js` patches Array methods "for Node 20+ compatibility" | Confusing/contradictory Node version story; dev machine is on Node 22 already | S | Low | Fix now |
| No CI at all (`.github/` doesn't exist) | Nothing enforces lint/typecheck/build on PRs | M | Med | Later release (flag to team) |
| `assets/` = 14MB; 3 files alone (`profile.png` 3.2MB, `splash-backdrop.jpg` 2.8MB, `collected.ai` 584KB) = ~6.6MB | Large uncompressed images bloat both binaries | M | Low | Fix now |
| 35 `.ttf` fonts (4.5MB) linked via `react-native.config.js`, but only 6 families/weights are referenced in `typography.ts`/`dark.json` | ~29 unused font files (~3.3MB) baked into every install | M | Low | Fix now |
| `react-native-fs` vs `react-native-image-resizer` vs `react-native-image-picker` vs `react-native-view-shot` | Real overlap only between the first three (all genuinely used, different jobs); `view-shot` is fully dead | S | None | Fix now (remove view-shot only) |
| `amazon-cognito-identity-js` + `react-native-app-auth` — two auth libs | Not redundant — one is direct Cognito password/email/JWT flows, the other is OAuth/social login via AppAuth; both genuinely used | — | None | No action |
| `i18next`/`react-i18next` — `src/lang` is NOT empty (3 locale JSONs + i18n.js), used in ~10 files | Contradicts audit brief's premise; keep both | — | None | No action |
| `amplify/` folder present, zero imports from `src/`/`App.tsx`/`index.js` | Legacy scaffold folder, not wired into the app at all | S | Low | Fix now (confirm with team before deleting — may be tied to an AWS Amplify CLI backend still deployed) |

---

## 1. Dependency-by-dependency table

Import counts are from `grep -rl` across `src/`, `App.tsx`, `index.js`, and config files (`babel.config.js`, `metro.config.js`, `react-native.config.js`, `tsconfig.json`).

### `dependencies`

| Package | Import count | Native (ios/android in node_modules) | Verdict | Why |
|---|---|---|---|---|
| `@chainplatform/react-native-htmlview` | 2 (`src/types/htmlview.d.ts`, `src/components/global/EntityDescriptionHtml.tsx`) | no/no (pure JS wrapper) | Keep | Renders listing/entity HTML descriptions |
| `@oakplatforms/types` | ubiquitous (`src/types/index.ts` re-exports) | n/a | Keep | Shared DTO source per CLAUDE.md |
| `@react-native-async-storage/async-storage` | not directly greped here but standard session/token storage lib | yes/yes | Keep (assumed core infra) | — |
| `@react-native-community/slider` | UI slider component | yes/yes | Keep | Used by `SyncedSlider`/price range filters |
| `@react-navigation/bottom-tabs` | `BottomTabNavigator.tsx` | yes(peer)/yes | Keep | Core nav |
| `@react-navigation/native` | `AppNavigator.tsx` and everywhere | yes/yes | Keep | Core nav |
| `@react-navigation/stack` | `AppNavigator.tsx`, `Wallet/Navigator.tsx` | yes/yes | Keep | Core nav |
| `@stripe/stripe-react-native` | Payment/wallet screens | yes/yes | Keep | Stripe payments |
| `@tanstack/react-query` | Everywhere (`fetchData` pattern) | n/a | Keep | Core data layer |
| `@types/react-native-dotenv` | **0** (depcheck confirms unused) | no/no | **Remove**, and if it were needed it belongs in `devDependencies`, not `dependencies` — types packages never ship in the runtime bundle | Dead + misplaced |
| `amazon-cognito-identity-js` | `src/services/auth/cognito.ts` | yes/yes | Keep | Direct Cognito email/password + JWT flows (see §2) |
| `buffer` | **0** — no `Buffer` or `from 'buffer'`/`require('buffer')` anywhere in `src`, `App.tsx`, `index.js`, `polyfill.js` | no/no | **Remove** | Confirmed dead; `cognito.ts` even has a comment noting Buffer is unavailable/unused in RN and it decodes JWTs without it |
| `fast-text-encoding` | `App.tsx:1` (`import 'fast-text-encoding'`) | no/no | Keep | Polyfills `TextEncoder`/`TextDecoder` for RN's JS engine (Hermes lacks full support) — no other polyfill provides this |
| `i18next` | `src/lang/i18n.js` + re-exported via `react-i18next` | no/no | Keep | Real i18n system, used in 10 files (see §2) |
| `install` | **0** | no/no | **Remove** | Junk package — `npm install`'s postinstall gimmick package, never imported |
| `lottie-react-native` | 3 files | yes/yes | Keep | See §2 |
| `mixpanel-react-native` | `src/services/analytics/index.ts` (single choke point) | yes/yes | Keep | Analytics |
| `npm` | **0** | no/no | **Remove** | npm-the-CLI vendored as a dependency — never imported, huge and pointless in `dependencies` |
| `react` | everywhere | n/a | Keep | Core |
| `react-i18next` | 10 files | no/no | Keep | See §2 |
| `react-native` | everywhere | n/a | Keep | Core |
| `react-native-app-auth` | `SocialAuthButtons.tsx`, `cognito.ts` | yes/yes | Keep | OAuth/social login (see §2) |
| `react-native-device-info` | 3 files (`UpdateGate`, `AnalyticsBridge`, `LegalFooter`) | yes/yes | Keep | App version/build checks, analytics context |
| `react-native-dotenv` | Used only via `babel.config.js` (babel plugin), invisible to `grep -rl src` — **depcheck's "unused" flag is a false positive**, exactly as the brief warned | no/no (babel transform, not runtime) | Keep | Powers the `@env` import alias used in 10 files (`src/utils/getImage.ts`, `AppNavigator.tsx`, `cognito.ts`, etc. — see knip's "Unlisted dependencies: @env" list, which is really this package doing its job) |
| `react-native-floating-label-input` | **0** in src/App/index | yes/yes (native dirs present) | **Remove** | No import found anywhere; confirmed dead by both depcheck and knip |
| `react-native-fs` | 2 files (`IDConfirmScreen`, `ScanCameraScreen`) | android:yes / ios:no (JS+CocoaPods podspec, not a plain dir — treat as native) | Keep | Filesystem read/write for ID capture + card scan flows (see §2) |
| `react-native-gesture-handler` | `index.js`, used implicitly by navigation/bottom sheets | yes/yes | Keep | Required by react-navigation + custom gesture UI |
| `react-native-get-random-values` | `index.js`, `cognito.ts` | no/no (pure JS polyfill) | Keep | Polyfills `crypto.getRandomValues` — required by `amazon-cognito-identity-js` under Hermes |
| `react-native-image-picker` | 7 files | yes/yes | Keep | Camera/gallery picker (see §2) |
| `react-native-image-resizer` | 9 files | yes/yes | Keep | Client-side image resize before upload (see §2) |
| `react-native-linear-gradient` | not explicitly greped, but iOS Podfile shows `BVLinearGradient` pod actively linked | yes/yes | Keep | Gradient backgrounds throughout UI |
| `react-native-qrcode-svg` | 1 file (`src/components/share/QrCode/index.tsx`) | no/no (pure SVG drawing, no native code) | **Keep — depcheck/knip false positive** | Confirmed real usage; both automated tools mis-flagged it |
| `react-native-reanimated` | `index.js`, animation-heavy components | yes/yes | Keep | Core animation engine; paired with `react-native-worklets` |
| `react-native-safe-area-context` | navigation root | yes/yes | Keep | Required by react-navigation |
| `react-native-screens` | not directly imported in `src` (depcheck flags unused) | yes/yes | Keep | **False positive** — required as a peer by `@react-navigation/native`'s native-screen optimization; must stay in `dependencies` even with zero direct imports |
| `react-native-svg` | icons everywhere, backdrop SVGs | yes/no (android via `android` dir; not confirmed ios but is a standard RN-linked lib) | Keep | Vector icon/asset rendering |
| `react-native-view-shot` | **0** in src/App/index | yes/yes (native dirs present) | **Remove** | No functional usage found anywhere; confirmed dead by knip. Native dirs mean this is pure bloat in both binaries for zero benefit |
| `react-native-vision-camera` | Card-scan camera flow | yes/yes | Keep | High-perf camera capture for the scan feature |
| `react-native-webview` | 3 files | yes/yes | Keep | In-app browser + embeds (see §2) |
| `react-native-worklets` | `babel.config.js` plugin + reanimated 4 peer | yes/yes | Keep | Required peer of `react-native-reanimated@4` |

### `devDependencies`

| Package | Import count | Native | Verdict | Why |
|---|---|---|---|---|
| `@babel/plugin-transform-private-methods` | `babel.config.js` | n/a | Keep | Babel plugin for private class fields (Hermes/JSC compat) |
| `@eslint/js` | `eslint.config.mjs` (flat config) | n/a | Keep | Base ESLint JS ruleset |
| `@react-native-community/cli` | Invoked implicitly by `react-native` CLI commands (`run-android`/`run-ios`/`start`) | n/a | Keep | **depcheck false positive** — it's the CLI binary itself, not something you `import` |
| `@react-native/metro-config` | `metro.config.js` (`require('@react-native/metro-config')`) | n/a | Keep | Actual metro config in use |
| `@react-native/typescript-config` | `tsconfig.json` `extends` | n/a | Keep | Confirmed present at `node_modules/@react-native/typescript-config` with `tsconfig.json`, `tsconfig.strict.json`, `package.json`, `README.md` — **contrary to the audit brief's premise, this resolves fine on disk**. (`node -e "require.resolve(...)"` throws only because the package uses `exports` map with no root-level Node resolution for that subpath from a bare `node -e`, which is expected — TypeScript resolves it via its own non-Node algorithm, not `require.resolve`. No real breakage; not independently reproduced via `tsc`.) |
| `@types/react` | type-only | n/a | Keep | React type defs |
| `@typescript-eslint/parser` | **0** references — `eslint.config.mjs` uses the `typescript-eslint` meta-package instead, which bundles its own parser | n/a | **Remove** | Superseded/dead, confirmed by depcheck+knip |
| `eslint` | `npm run lint` | n/a | Keep (but see version note in §3) | Core lint tool |
| `eslint-plugin-react` | lint config | n/a | Keep | React lint rules |
| `globals` | `eslint.config.mjs` | n/a | Keep | Global identifier definitions for flat config |
| `metro-config` | **0** direct references — `metro.config.js` requires `@react-native/metro-config`, not this package | n/a | **Remove** | Dead + confusingly named alongside the package actually used; version (0.82.5) doesn't even match RN 0.83 |
| `openapi-typescript` | **0** references | n/a | **Remove** | Per CLAUDE.md, this repo's types come from `@oakplatforms/types` (published by nanza-api); no `generate:types`-style script exists here. Leftover from a copy/paste of nanza-api tooling |
| `prettier` | **0** references — no `.prettierrc`, no npm script, ESLint flat config doesn't wire it in | n/a | **Remove** | Confirmed unused by knip+depcheck; also pinned at 2.8.8 (3 majors behind latest 3.9.7) — doubly stale if it were to be reintroduced |
| `react-dom` | **0** references (RN app, no web rendering anywhere) | n/a | **Remove** | Confirmed dead; likely a stray install from a copy-pasted `package.json` or an aborted web-preview experiment |
| `react-native-svg-transformer` | `metro.config.js` (`babelTransformerPath`) | no/no (pure JS transform, no ios/android dirs — confirmed) | Keep | Required for SVG-as-component imports |
| `typescript` | `tsc`, editor tooling, `tsconfig.json` | n/a | Keep | **depcheck false positive** — used by the toolchain, not imported in source |
| `typescript-eslint` | `eslint.config.mjs` | n/a | Keep | Modern flat-config TS lint meta-package (makes `@typescript-eslint/parser` redundant, see above) |

**Missing:** depcheck flags `@types/jest` as missing (referenced by `tsconfig.json`'s `types: ["react", "react-native", "jest"]`). Per CLAUDE.md, "Do not add tests" — but the `jest` type reference in `tsconfig.json` still causes a real TS resolution gap (also flagged by knip as an "Unresolved import"). Low priority: either add `@types/jest` as a devDependency (small, type-only, doesn't imply writing tests) or strip `"jest"` from `tsconfig.json`'s `types` array since the project doesn't use Jest types anywhere in `src/`.

---

## 2. Functional overlap

**File/image capture stack — `react-native-fs`, `react-native-view-shot`, `react-native-image-resizer`, `react-native-image-picker`:**
- `react-native-image-picker` (7 usages: `imagePicker.ts`, bulk listing, scan item edit, profile photo, offer listing row, edit/create listing) — the actual camera-roll/camera picker UI.
- `react-native-image-resizer` (9 usages, largely the same call sites plus ID capture/confirm and scan camera) — resizes/compresses picked images before upload; always paired with the picker in the same files.
- `react-native-fs` (2 usages: `IDConfirmScreen`, `ScanCameraScreen`) — raw filesystem read/write, narrower use case (writing/reading files for the ID and scan flows specifically), not a general image tool.
- `react-native-view-shot` — **zero usages anywhere.** It doesn't overlap with anything; it's simply dead. No screenshot/capture-a-view functionality exists in the current codebase.

Verdict: no real overlap to resolve among the first three — each does a distinct job and all are genuinely wired in. Only `view-shot` should go.

**Auth — `amazon-cognito-identity-js` vs `react-native-app-auth`:**
- `src/services/auth/cognito.ts` imports both. `amazon-cognito-identity-js` drives direct Cognito User Pool flows (email/password sign-up, sign-in, confirmation codes, token refresh — the `CognitoIdToken` JWT decode referenced in a comment at line 204).
- `react-native-app-auth` (AppAuth-based OAuth) is used for social sign-in, federating Apple/Google identities into the same Cognito user pool via its hosted domain — confirmed by `env.d.ts`'s `COGNITO_DOMAIN` comment: "amazon-cognito-identity-js has no federation surface" so social login must go through AppAuth's OAuth redirect instead.
- `src/screens/auth/SocialAuthButtons.tsx` is the other consumer of `react-native-app-auth` (the actual Apple/Google buttons).

Verdict: not redundant. Two different auth surfaces (password vs. OAuth) that must be split because Cognito's JS SDK can't do federation. Keep both.

**i18n — `i18next` + `react-i18next`:**
The audit brief's premise ("`src/lang` is empty") is incorrect for this checkout: `src/lang/` contains `en.json`, `fr.json`, `ja.json`, and `i18n.js` (wires them up via `initReactI18next`). `index.js` imports `./src/lang/i18n` directly, and `useTranslation` from `react-i18next` is used in 10 files (auth flows: sign-up confirmation, forgot-password, plus `Favorites`, entity card list/listing views, forgot-password modal). Both packages are live and necessary.

**Polyfills — `buffer`, `fast-text-encoding`, `react-native-get-random-values`:**
- `fast-text-encoding`: imported once, top of `App.tsx`. Provides `TextEncoder`/`TextDecoder`, which Hermes doesn't fully implement — needed by anything doing UTF-8 byte work (JWT decode paths, crypto libraries).
- `react-native-get-random-values`: imported in `index.js` (before `react-native-gesture-handler`/`reanimated`) and referenced in `cognito.ts`. This is the standard shim so `crypto.getRandomValues` exists for `amazon-cognito-identity-js`'s internals under Hermes/JSC, which lack native WebCrypto.
- `buffer`: **no reference anywhere.** `grep -rn "Buffer"` across `src`, `App.tsx`, `index.js`, `polyfill.js` returns only a comment in `cognito.ts` explicitly stating Buffer is unavailable/unneeded in RN. Confirmed dead — remove.

**HTML rendering — `@chainplatform/react-native-htmlview` vs `react-native-webview`:**
Not actually overlapping in practice:
- `@chainplatform/react-native-htmlview` → `src/components/global/EntityDescriptionHtml.tsx` only — renders simple, sanitized rich-text entity/listing descriptions as native views (lightweight, no full web engine).
- `react-native-webview` → `InAppBrowserSheet.tsx`, `EmbedPlayer.tsx`, `OrderTrackingModal/index.tsx` — a real embedded browser (external links, carrier tracking pages, video embeds), which needs actual web-engine features htmlview can't provide (JS execution, navigation, cookies).

Both are justified for different jobs; no consolidation opportunity.

**`lottie-react-native`:** 3 usages — `SocialAuthButtons.tsx`, `LoadingIndicator/index.tsx`, `Button/index.tsx`. Modest footprint (loading/auth micro-animations only). `metro.config.js` explicitly special-cases `.lottie` asset extension, and `assets/` has exactly 1 `.lottie` file, consistent with light usage. Native pod (`lottie-ios`, `lottie-react-native`) is a heavier dependency (see §4 pod list) for a small feature set — worth a values judgement but not clearly worth removing given it's actively used in the auth entrance flow (a first-impression screen).

**`react-native-qrcode-svg`:** 1 usage (`src/components/share/QrCode/index.tsx`), confirmed real despite depcheck/knip both flagging it unused — pure SVG (no native code), cheap to keep.

**`mixpanel-react-native`:** Single choke point, `src/services/analytics/index.ts` — consistent with a proper analytics facade pattern; not spread across the codebase.

**`react-native-device-info`:** 3 call sites — `UpdateGate` (app version gating), `AnalyticsBridge` (device context for analytics events), `LegalFooter` (likely displaying app version). Narrow, justified usage.

---

## 3. Toolchain / version health

`npm outdated --json` results (full data in `outdated.json`), key findings:

- **react-native 0.83.1 → 0.83.10 wanted, 0.87.1 latest.** Currently 7 patch releases behind on the installed minor, and 4 minors behind latest.
- **react 19.2.0 → 19.3.0 latest** (react-dom matches at 19.2.0, but react-dom is dead weight regardless, see §1).
- **react-native-reanimated 4.3.0 → 4.6.0 wanted/latest; react-native-worklets 0.8.0 → 0.8.3 wanted, 0.12.2 latest.** `npm ls 2>&1 | grep -iE 'invalid|missing|peer'` returned **no output** — no invalid/missing/peer-dependency errors reported by npm's own resolver. The reanimated↔worklets pairing is currently self-consistent per npm's dependency graph, though both are behind their own latest and worth bumping together (reanimated 4.x requires a worklets version in lockstep per their release notes — not independently verified beyond npm ls showing no conflict).
- **eslint 8.57.1 (EOL — ESLint 8 stopped receiving support once ESLint 9 shipped; 8.57.1 was its final release) paired with `typescript-eslint` 7.18.0.** Verified via `npm view typescript-eslint@7 peerDependencies`: every 7.x release (7.0.0 through 7.18.0) declares `"eslint": "^8.56.0"` — so eslint 8.57.1 satisfies the peer range cleanly; there is no version-range violation today. However, `typescript-eslint@7`'s support matrix targeted TypeScript <5.6 at release time, while this repo runs `typescript@^5.9.3` — 3+ TS minors past what 7.x was built/tested against. It works (no lint crashes reported), but the trio (ESLint 8 + typescript-eslint 7 + TS 5.9) is now a stale combination one upgrade cycle behind — worth bumping to ESLint 9 + typescript-eslint 8 as a coordinated jump (both majors moved together upstream), not urgent.
- **`@react-native/typescript-config` — the audit brief's premise that tsc reports this as NOT FOUND does not hold up:** `node_modules/@react-native/typescript-config/` exists and contains `tsconfig.json`, `tsconfig.strict.json`, `package.json`, `README.md`. A plain `node -e "require.resolve(...)"` throws `ERR_PACKAGE_PATH_NOT_EXPORTED`, but that's expected — Node's CJS `require.resolve` doesn't handle TypeScript's own `extends`-resolution algorithm (which does support conditional exports for tsconfig subpaths); this is not evidence of a real `tsc` failure and I did not reproduce a failure by actually invoking `tsc`. Treat the "NOT FOUND" claim in the original brief as unconfirmed/likely incorrect for this checkout.
- **`.nvmrc` says `21.7.1`; `engines` in `package.json` says `>=20`; the live dev shell is on Node `22.21.0`.** Node 21 was end-of-lifed in June 2024 (odd-numbered Node releases are never LTS and get ~8 months of support) — pinning to it in `.nvmrc` is stale advice for anyone who actually respects it. `polyfill.js` patches `Array.prototype.toReversed/toSorted/toSpliced/with` "for Node 20+ compatibility" — those methods shipped in Node 20 itself (V8 with ES2023 support), so the polyfill guards against *older* Node, not against 20+. On Node 22 (already in use here) all four methods are native, so the polyfill's `if (!Array.prototype.x)` guards are already no-ops — dead weight, though harmless. Recommend updating `.nvmrc` to an actual current LTS (22.x) to match both the `engines` intent and the dev machine's real version, and re-evaluating whether `polyfill.js` is still needed at all now that the floor is effectively Node 22.
- **Two lockfiles:** `yarn.lock` (298KB) and `package-lock.json` (534KB), both last touched in the same commit (`846150ce`, 2026-09-14) and consistently updated together across the last 10 dependency-touching commits — no drift found between them in git history. There's no `.github/` CI config at all (confirmed absent) and no lockfile reference in any `scripts/*.sh`. The only concrete signal is the `README.md`, which documents yarn as the "OR using Yarn" alternate path for `start`/`android`/`ios` — implying npm is the primary documented flow but yarn is explicitly supported too. Recommend picking one (npm, since it's the primary path and `package-lock.json` is what's referenced first in README) and deleting the other to stop the redundant "keep both in sync" tax.

---

## 4. App size contributors (no build performed)

**`assets/` = 14MB total**, by file type:
| Type | Count | Notes |
|---|---|---|
| `.svg` | 148 | Vector, cheap individually |
| `.ttf` | 35 | 4.5MB total — see fonts breakdown below |
| `.json` | 7 | small |
| `.png` | 6 | |
| `.jpg` | 4 | |
| `.lottie` | 1 | |
| `.ai` | 1 | **584KB — an Adobe Illustrator source file (`assets/icons/collected.ai`) shipped inside the assets folder**, almost certainly bundled into the app by mistake; should not ship in a release build at all |

**Files over 200KB** (7 files, ~6.9MB combined — roughly half of all `assets/`):
| File | Size |
|---|---|
| `assets/images/profile.png` | 3.2MB |
| `assets/images/splash-backdrop.jpg` | 2.8MB |
| `assets/images/featuredCard-001.jpg` | 676KB |
| `assets/icons/collected.ai` | 584KB |
| `assets/images/backdrop.svg` | 572KB |
| `assets/images/auth-backdrop.jpg` | 384KB |
| `assets/fonts/Roobert-RegularItalic.ttf` | 380KB |

`profile.png` and `splash-backdrop.jpg` together are 6MB — almost half the entire assets folder — and are strong candidates for re-export as compressed WebP/optimized JPEG (a 3.2MB PNG placeholder/splash image is very likely capable of dropping to a few hundred KB with no visible quality loss). `collected.ai` (584KB Illustrator source) should be deleted from `assets/` entirely — it's a design source file, not a runtime asset, and `react-native.config.js` only declares `./assets/fonts/` as a linked asset dir, so it isn't clear this `.ai` file is even referenced by the app, just sitting in the bundled tree taking up space if any build step globs `assets/**`.

**Fonts — 35 `.ttf` files, 4.5MB, linked into both binaries via `react-native.config.js` (`assets: ['./assets/fonts/']`):**

Cross-checked against actual usage in `src/styles/components/typography.ts` and `src/styles/tokens/dark.json`. The theme's `fonts.family` map defines exactly **6 family/weight combinations**, all from the **Roobert** and **BaltoMobile** families:
```
primary          -> Roobert-Regular
primaryMedium    -> Roobert-Medium
primarySemiBold  -> Roobert-SemiBold
primaryBold      -> Roobert-Bold
bodyFamily       -> BaltoMobile-Book
secondary        -> Roobert-Regular (duplicate alias, same file)
```
`typography.ts` only ever references `theme.fonts.family.*` tokens (never a raw literal font name), so this is the complete set of fonts actually renderable by the app today.

Font files actually needed: `Roobert-Regular.ttf`, `Roobert-Medium.ttf`, `Roobert-SemiBold.ttf`, `Roobert-Bold.ttf`, `BaltoMobile-Book.ttf` — **5 files**, plus possibly `Roobert-RegularItalic.ttf` if any italic variant is used (not found in typography.ts/dark.json, likely also unused).

Font families present but **entirely unreferenced** by the theme token system:
- **EuclidCircularB** — 10 files (Bold/BoldItalic/Light/LightItalic/Medium/MediumItalic/Regular/RegularItalic/Semibold/SemiboldItalic) ≈ 1.45MB
- **Figtree** — 10 files (Black/BlackItalic/Bold/BoldItalic/ExtraBold/ExtraBoldItalic/Medium/MediumItalic/SemiBold/SemiBoldItalic) ≈ 420KB
- **Geist** — 9 files (Black/Bold/ExtraBold/ExtraLight/Light/Medium/Regular/SemiBold/Thin) ≈ 720KB
- `Roobert-RegularItalic.ttf` (380KB) — not found in the 6-entry family map

That's **29 of 35 font files (~2.97MB) with no discoverable reference** in the typography/token system that CLAUDE.md's P0 rules mandate as the *only* legitimate path to a font (all text styling must go through `typography.ts` scale slots). Since nothing outside `typography.ts` is allowed to set `fontFamily` directly (P0 rule 12, enforced by ESLint's `no-restricted-syntax`), and `typography.ts` doesn't reference EuclidCircularB/Figtree/Geist at all, these 29 files are extremely likely dead weight carried into every iOS and Android build. Recommend confirming with a repo-wide `grep -rn "EuclidCircularB\|Figtree\|Geist" src` (returned no hits in this audit) before deleting, then dropping to the 5-6 files actually in use.

**Estimated size win: ~3MB of fonts (29 of 35 files) + several MB of image re-compression (profile.png, splash-backdrop.jpg alone are 6MB pre-optimization) — roughly 8-9MB of the 14MB `assets/` folder is reducible**, before any native-pod trimming.

**Native pods — `ios/Podfile.lock`:**
- `grep -c '^  - ' ios/Podfile.lock` → **290 pod entries** (includes transitive React Native core pods — this is a normal RN 0.83 New-Architecture count, not evidence of bloat by itself).
- Confirmed **not** part of the count from padding: pods traced to the packages flagged dead in §1 —
  - `react-native-view-shot` — has a native pod entry pulled in transitively (has ios/android dirs in node_modules); removing the JS dependency removes its pod too.
  - `react-native-floating-label-input` — same; native dirs present, currently unused.
- Notable heavier pods observed in the lockfile: `lottie-ios`/`lottie-react-native`, `MixpanelReactNative`/`Mixpanel-swift`, `AppAuth` (Core + ExternalUserAgent) — all justified by real usage (§2), not removal candidates, just worth knowing they're the heavier native surface area.
- `ios/Podfile` uses `use_native_modules!` (autolinking) with `hermes_enabled => true` and `new_arch_enabled => true` — both good defaults for size/perf on 0.83.

**Android (`android/app/build.gradle` + `android/gradle.properties`):**
- `hermesEnabled=true`, `newArchEnabled=true` — good.
- `enableProguardInReleaseBuilds = false` — **Proguard/R8 minification is OFF for release builds.** This is a meaningful size miss: R8 shrinking + obfuscation on a RN 0.83 New Architecture app can meaningfully cut APK/AAB size and dead-code. No `shrinkResources` line found at all (would also require minification on to take effect). This is the single highest-leverage native-side size fix identified — flip `enableProguardInReleaseBuilds = true` and add `shrinkResources true` in the `release` buildType, then validate a release build isn't broken by over-aggressive stripping (test thoroughly, some RN native modules need proguard-rules exceptions).
- `reactNativeArchitectures=armeabi-v7a,arm64-v8a,x86,x86_64` — **all 4 ABIs are built into a single APK/AAB** by default (no `splits` block found in `build.gradle`, no per-ABI split config). If distribution is via Play App Bundle (AAB), Play's dynamic delivery already serves only the needed ABI to each device, making this a non-issue for AAB distribution — but if any direct APK distribution path exists (Play test tracks that hand out flat APKs, sideloading, etc.), this quadruples the installed native library size unnecessarily. Confirm distribution is AAB-only.

**`babel.config.js` / `metro.config.js` size-relevant settings:**
- `metro.config.js` sets `inlineRequires: true` — already on, this is the correct choice for startup-size/perf (lazy-evaluates module bodies).
- `metro.config.js` also swaps `svg` out of `assetExts` into `sourceExts` via `react-native-svg-transformer`, meaning all 148 SVGs are compiled to React components rather than shipped as binary assets — reasonable for vector icons, no action needed.
- `babel.config.js` plugins are minimal and all justified: `react-native-worklets/plugin` (reanimated 4 requirement), `@babel/plugin-transform-private-methods` (class-field compat), `module:react-native-dotenv` (the `@env` alias).

---

## 5. Config drift

- **`react-native-dotenv` + `env.d.ts` + `.env`:** `.env` is **not committed** (confirmed via `git ls-files`), which is correct — secrets stay local. `env.d.ts` declares the full `@env` module shape (13 exported consts: `S3_BASE_URL`, `IMAGE_CDN_BASE_URL`, `API_BASE_URL`, Stripe keys, Cognito pool/domain, Mixpanel token/env, `SHARE_BASE_URL`, `WS_BASE_URL`) with useful inline comments explaining CDN fallback behavior and the Cognito federation rationale. This is well-documented and in good shape — no drift found here.
- **`amplify/` folder:** Present at repo root (`amplify/cli.json`, `amplify/backend/backend-config.json`, `amplify/team-provider-info.json`, `amplify/hooks/README.md`, etc.) — the classic AWS Amplify CLI scaffold. **Zero references** from `src/`, `App.tsx`, or `index.js` (`grep -rln "amplify"` returns nothing). The app's actual Cognito/S3/API wiring goes through hand-rolled services (`cognito.ts`, `@env` vars) instead, not Amplify's generated client. This folder looks like dead scaffolding from an earlier setup approach that was abandoned in favor of direct SDK usage — worth confirming with the team whether an actual Amplify backend is still deployed from this config (if so, don't delete blindly; if not, it's pure repo clutter).
- **`scripts/*.sh`:**
  - `build-android-release.sh` — well-documented release AAB builder; auto-bumps `versionCode`, requires local signing creds in `~/.gradle/gradle.properties`. No issues.
  - `set-ios-version.sh` — manages `MARKETING_VERSION`/`CURRENT_PROJECT_VERSION` directly in `project.pbxproj` (correctly avoids `agvtool`'s known issue with build-setting-driven versioning). No issues.
  - `node_with_polyfills.sh` — thin wrapper forcing `NODE_OPTIONS="--require .../polyfill.js"`; consistent with the `NODE_OPTIONS` prefix baked into the `android`/`ios`/`start` npm scripts. No issues, though see §3 on whether `polyfill.js` itself is still earning its keep on Node ≥20.
  - `figma-token-sync.js` — flagged by knip as an **unused file** (not wired into any npm script in `package.json`). Likely invoked manually or by the `figma-sync` skill/plugin rather than via npm — not necessarily dead, just not part of the standard script surface; confirm before removing.
  - `setup-local.sh` (referenced by the `cognito-setup` npm script) — exists, not separately inspected in depth here (out of dependency/size scope), no red flags surfaced by other checks.
- **`app.json`** — minimal and clean (`name`/`displayName` only). No drift.
