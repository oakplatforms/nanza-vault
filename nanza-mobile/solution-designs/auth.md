---
tags:
  - nanza-mobile
  - solution-design
  - auth
  - legal
---

# Auth

How someone gets into Nanza: the two credential paths, the four surfaces they run on, and
the terms agreement that has to appear on both entry points.

## The shape: one interstitial, one screen flow, two sheets

Sign-up used to be a nine-case step machine living inside `AuthModal` — Email → MobileNumber
→ Password → ConfirmationCode → Username → Agreement → ProfilePicture → Welcome, plus Login.
That is retired (Aug 2026). The steps were split the way they were because a **sheet** has a
height budget; full screens have no such excuse, so the flow was rebuilt as screens and the
modal kept only the job it was best at.

| Surface | Kind | Role |
| --- | --- | --- |
| `components/modals/AuthModal` | Sheet | **Marketing interstitial only.** "Want to join?" — a guest can dismiss it and keep browsing. |
| `screens/auth/SignUpScreen` | Root-stack screen | Step 1: email **and** password on one screen. |
| `screens/auth/SignUpScreen/ConfirmationCode` | Root-stack screen | Step 2: the emailed code. |
| `components/modals/LoginModal` | Sheet | Log in — one form, one submit, so it stays a sheet. |
| `components/modals/CompleteProfileModal` | Locked sheet | The username step, over Home. |

`AuthScreenLayout` is the shared frame for the screens and draws its own back circle, so the
flow keeps one chrome throughout (`headerShown: false` on both).

Two deliberate details:

- **`AuthModal` kept its component name and its `visible`/`onClose` contract**, so every
  existing caller — TopHeader's Log In pill, JoinChatFooter, the share cards — keeps working
  untouched. `initialStep` survives as an ignored hint.
- **Navigation is deferred to `onDismissComplete`, not fired on press.** The sheet must finish
  animating out before the stack pushes, or the push lands under a sheet still on screen.
- **`gestureEnabled` is off on mid-flow steps.** A swipe-back after the account exists would
  re-open a step whose server call already succeeded.

### CompleteProfileModal is self-gating, and that is the design

It renders whenever the session says *signed in + has an account + no `profile.username`* —
it is never pushed by a flow. That single condition covers both the social sign-in that just
finished **and** the hole where someone kills the app mid-completion and boots straight back
to Home with a live session. The sheet simply reappears until the username exists. It is
deliberately inescapable (no backdrop tap, no swipe-down, no close affordance): a Nanza
account is unusable without a username. Completion is implicit — `refreshCurrentUser()`
re-derives the gate and the sheet unmounts itself.

## Two credential paths

**Email + password** goes straight through `amazon-cognito-identity-js`. `SignUpScreen` does
both server calls the old two steps did separately: `authService.checkForUser` first, then
Cognito `signUp`. Order matters — the check has to clear before anything is created, so a
taken email reads as a field error rather than surfacing later as a Cognito
"User already exists".

**Social (Apple / Google)** cannot. Federating a provider identity into a **user pool** has
no direct token-exchange API, and `amazon-cognito-identity-js` has no federation surface at
all. So `signInWithProvider` runs an OAuth **authorization-code flow with PKCE** against the
pool's *hosted domain* (`COGNITO_DOMAIN`, host only — no scheme, no trailing slash) through
`react-native-app-auth`, then persists the returned tokens with the same `persistSession`
the password path uses. Both paths converge on one session.

Consequences worth remembering:

- The provider's native sheet still appears, but iOS adds a one-time
  "…wants to sign in using amazoncognito.com" consent prompt.
- **Apple shows on both platforms.** It was iOS-only while the plan was a native Apple SDK
  exchange; since the switch to hosted OAuth, Android runs the same `authorize()` through a
  Custom Tab and Cognito handles Apple's web sign-in server-side.
- **A cancel is not an error.** Dismissing the sheet rejects; alerting on that would punish
  backing out. Only real failures alert.
- **PostConfirmation *does* fire for federated users** — verified on device 2026-08-24,
  correcting an earlier claim in this vault that it was skipped. The app User/Account row is
  created, so no Pre-Token handler is needed. What the row lacks is a username (and possibly
  an email, if the provider mapping gaps), which is exactly what CompleteProfileModal gates on.

`SocialAuthButtons` renders the providers above the fields on sign-up, and `compact`
(icon-only, side by side) inside LoginModal — on log-in the buttons are a shortcut past the
password, not the offer being sold, and "Sign up with…" copy would be wrong there.

### Native wiring the OAuth redirect depends on

Four pieces, all required; miss one and the sign-in promise never settles:

- `AppDelegate` conforms to `RNAppAuthAuthorizationFlowManager` and exposes
  `authorizationFlowManagerDelegate`.
- `openURL:` gives that delegate **first refusal** on the `nanza://auth` callback, falling
  through to `RCTLinkingManager` only if unclaimed — otherwise share deep links break.
- Android: `manifestPlaceholders = [appAuthRedirectScheme: "nanza"]` in `app/build.gradle`.
- iOS `Info.plist` carries the `nanza` URL scheme; `COGNITO_DOMAIN` is set in `.env`.

## The terms agreement

App Review (Guideline 1.2, user-generated content) requires the agreement before **both**
registering and logging in. `components/auth/AuthTermsNotice` owns it so the wording cannot
drift between the two entry points — one sentence, one component, an `action` prop switching
only the verb: *"By creating an account…"* on sign-up, *"By continuing…"* on log-in, because
someone logging in isn't creating anything.

It replaced a plain `<Text>` that named the two documents without linking them — naming is
not enough; the documents have to be **reachable**.

**It opens the hosted web pages through `InAppBrowserSheet` rather than navigating.** That
started as a constraint: it first lived inside `AuthModal`'s RN `<Modal>`, which portals to
its own view hierarchy, so a pushed screen landed *behind* the sheet and read as a dead tap.
Sign-up is now a full screen where navigation would work — but LoginModal is still a sheet,
so the browser path is the one road that serves both. Switching to navigation is a live
option (the native screens and routes below already exist), just not one taken yet.

## The legal documents

Native, deep-linkable, and separate from the notice that links to them:

- `screens/legal/TermsOfServiceScreen` → `terms`
- `screens/legal/PrivacyPolicyScreen` → `privacy-policy`

Both are thin `PageLayout` hosts wrapping a body component (`components/legal/*Body`) —
the same shape as the older `ConditionsOfUseScreen`. Registered on the root stack with
`linkingConfig` entries, so a marketing email or web link opens the document in-app.

## Typography

The notice uses three slots added to `typography.ts` rather than restyling shared ones:
`authTermsText` (the `legal` slot — the quietest text in the flow), `authTermsLink` (same
slot, underlined, `textMuted`), and `authSecondaryActionText`. That last one is deliberately
**not** a change to `textLink`, which is shared with ConfirmationCode, LoginForm and
ForgotPassword and keeps its own colour.

## Known gaps

- **`fr` / `ja` are stale.** `emailScreen.header` was changed to "Join the Flesh and Blood
  community" in English only; the other locales still read "What's your email address?".
  Needs a translation pass.
- **`authScreens.legalText`** is now an unused style — the sentence it dressed was replaced
  by AuthTermsNotice, which brings its own slots.
- **`tsc` cannot run** repo-wide: `tsconfig.json` lists `"jest"` in `types` but `@types/jest`
  has never been declared in `package.json`. Pre-existing, unrelated to auth, but it means
  the P0 typecheck gate is currently unavailable to everyone.
