---
tags: [nanza-mobile, solution-design, plan, auth, signup]
---

# Auth: modal → full screens, + social sign-in — Plan

> **Status: BUILT & LIVE IN DEV (2026-08-25) — Apple and Google sign-in verified end-to-end;**
> originally planned 2026-08-22 (Skylar, voice session). Supersedes the modal-hosted
> sign-up flow. Pairs with [[social-auth]] (the Apple/Google client design, already
> written) and the infra half in `nanza-auth/solution-designs/social-identity-providers.md`.
> Follows the signed-out UX pass in [[open-posting-and-nav-swap#Sixth pass — the signed-out nav (2026-08-22, Skylar)]].

## The ask

The modal stops being the sign-up flow and becomes a **marketing interstitial** — one
salesy step that asks you to join or lets you carry on browsing. Everything after it
moves to **full screens**, and social sign-in lands at the top of the first one.

## Today

`AuthModal` (`src/components/modals/AuthModal/index.tsx`) is a nine-case step machine
(`Email → MobileNumber → Password → ConfirmationCode → Username → Agreement →
ProfilePicture → Welcome`, plus `Login`) rendered inside one `Modal`, with a shared
Nanza logo above `renderStep()`. The step components already live at
`src/screens/auth/SignUpScreen/*.tsx` — they are screens in everything but hosting,
which makes this move mostly a re-parenting job rather than a rewrite.

Callers mount `<AuthModal visible … initialStep>` locally (TopHeader at `Login`,
JoinChatFooter and the share cards at `Email`). There is no global auth-modal host.

## The target

> **Two modals survive.** Confirmed explicitly (Skylar, 2026-08-22): the marketing
> interstitial stays a sheet, and log in became one too. Only the three SIGN-UP steps
> are full screens.

### 1. The modal becomes one step only

Content: Nanza logo (kept), a headline + short body (placeholder copy for now), and
**two buttons**:
- **Sign up** → dismiss the modal, navigate to the sign-up stack.
- **Keep browsing** → dismiss, do nothing else.

Every existing `<AuthModal>` caller keeps working unchanged — same component, same
`visible` prop; only its content changes. `initialStep` collapses away (see open
decisions for the `Login` case).

### 2. Sign-up becomes three screens

Down from seven. (A fourth, skippable avatar step was built and then dropped —
Skylar, 2026-08-22: username is the end of the flow. Adding a photo stays available
from the profile.) Stepped rather than one long form, deliberately:

| # | Screen | Contents |
|---|--------|----------|
| 1 | **Create account** | Apple + Google buttons, an "or" divider, then email + password together. Agreement as fine print under the primary button. |
| 2 | **Confirm code** | The emailed code. |
| 3 | **Username** | With its server-side uniqueness check. Last step — completing it lands on `returnTo`/Main. |

**Why stepped, not one form:** the confirmation code cannot be filled until the email
has already been submitted, so a single form would strand the user mid-page; and
username needs a round-trip to validate. Those two facts force at least three
submissions, so the steps are real, not decorative.

**Merges:** Email + Password combine into screen 1 (they were two steps for no reason
once the page is full-height). `MobileNumber` drops out of the flow. `Agreement`
becomes fine print rather than a screen — the current norm, and it removes a step
that never collected input. `Welcome` folds into the post-completion navigation.

### 3. Social sign-in

Screen 1 carries **Sign in with Apple** and **Continue with Google** ABOVE the email
fields, with an "or" divider between. This is the placement [[social-auth]] already
specifies. That design's native-SDK decision (no Cognito Hosted UI, no browser bounce)
stands unchanged; this plan only fixes *where* the buttons live now that sign-up is a
screen rather than a modal step.

Note the ordering dependency: the buttons cannot function until the infra half is done
(Apple/Google registered as Cognito IdPs, plus the Pre-Token-Generation trigger that
provisions federated users — federated identities never fire
`PostConfirmation_ConfirmSignUp`). **The screens can ship before that; the buttons
cannot.** Build the layout, gate the buttons behind the credentials landing.

## Known risk

**The screens will look sparse.** A single input on a full-height screen is a lot of
empty space — Skylar flagged this while approving the structure. Accepted for now and
revisited once they're real on device. Adding banner/profile fields to the username
screen to fill it was considered and rejected the same session (it muddies one clear
ask per screen). If sparseness reads badly, the lever is richer per-screen framing
(headline + supporting copy + progress), not more inputs.

## Implementation record (2026-08-22, uncommitted)

Built same session. Specifics beyond the plan:

- **New**: `screens/auth/AuthScreenLayout.tsx` (the frame all three steps share — the
  shared Header, title block, scrolling fields, pinned actions), `styles/components/authScreens.ts`,
  `screens/auth/SocialAuthButtons.tsx`, `screens/auth/SignUpScreen/index.tsx` (the merged
  credentials step), `components/modals/LoginModal/index.tsx`.
- **`headerRow` keeps its height when back is hidden** — otherwise step 1 starts higher
  than the rest and the title jumps between screens.
- **`ConfirmationCode` was converted, not rewritten**: its Cognito error parsing (the
  PostConfirmation-lambda decoding, the code-mismatch/expired/limit family) is untouched.
  Only the shell and forward nav changed.
- **`ProfilePicture` was rewritten and then deleted the same day.** The original was
  broken (it navigated to an unregistered `Welcome` screen and never uploaded anything);
  the rewrite worked, but Skylar cut the step entirely — so `Username` is now the final
  screen and completes the flow. Its styles were removed with it.
- **Log in is a SHEET, not a screen** (`components/modals/LoginModal`). A `LogInScreen`
  was built first and removed the same day (Skylar): sign-up earns full screens because
  it is multi-step with a code round-trip and a uniqueness check, whereas log in is one
  form and one submit, so it never needs to enter the navigation stack. `LoginForm` is
  reused untouched — closer to an unwind than a rewrite — with its callbacks mapped to
  dismiss-then-navigate. There is deliberately no `LogIn` route in RootStackParamList.
- **A `height="auto"` sheet MUST wrap its children in a `ScrollView`.** Both new modals
  shipped without one and stretched far past their content (~300px of dead space under
  the buttons). `modalContentWrapper` carries `flex: 1`, and the auto variant alone
  doesn't stop the body expanding — every other auto-height sheet in the app
  (ForgotPasswordModal, ActionModal, WithdrawModal) has the ScrollView, which is why they
  size correctly. Worth knowing before writing the next one.
- **The close control is the shared `Header`** (`floating` + `showBackButton` +
  `backIcon="chevron-down-2"`), not a hand-rolled circle. Header owns the glyph box —
  every non-`chevron-left-3` back glyph is square at `icons.base` — so reusing it is what
  makes this control identical to the composer's and the bulk screens'. A local circle at
  `icons.sm` read visibly small and used the wrong glyph.
  - Header computes its own height as `insets.top + 66`, so the screen must NOT add
    `paddingTop: insets.top` as well — that doubles it. Mount `<Header>` as a bare first
    child with no wrapper and no offset, exactly as CreateListingsScreen and
    ScanItemEditScreen do; a `-gutter` lift was tried and removed, since matching those
    screens is the whole point.
- **`gestureEnabled: false` on the mid-flow screens** (code, username): a
  swipe-back after the account exists would re-open a step whose server call already
  succeeded. This is the old modal's `isLockedStep` rule, carried over.
- **AuthModal keeps its exact public contract** (`visible`/`onClose`/`returnTo`), so all
  23 call sites were untouched. `initialStep` narrowed from nine steps to
  `'Email' | 'Login'` — both existing callers pass `"Login"`, so both still compile.
  Navigation fires on `onDismissComplete`, not on press, or the push lands under a sheet
  still animating out.
- **Deleted**: `Email.tsx`, `Password.tsx`, `Agreement.tsx`, `MobileNumber.tsx`,
  `Welcome.tsx`.
- **Social buttons are layout-only** and say "coming soon" on press — see the ordering
  dependency above. Wiring them is: install the two native SDKs, add
  `signInWithApple`/`signInWithGoogle` to `services/auth/cognito.ts`, call them in
  `SocialAuthButtons`.
- Verified: tsc + eslint clean on all touched files (repo baseline errors unchanged).
  **NOT run on device** — no simulator this session.

## Social completion gate (2026-08-24, built)

Both signup paths now end at the same **completion screen** (`SignUpScreen/Username.tsx`):

- **Email flow**: unchanged — arrives with `userData.userId`, asks username only.
- **Social flow**: `SocialAuthButtons` fetches the user after the OAuth exchange and, if
  `account.profile.username` or `account.email` is missing, replaces to the completion
  screen instead of Main — WITHOUT `setIsSignedIn`, so the app shell never flips around a
  half-made account. The screen resolves its authId from `getCurrentUser()` when
  AuthContext is empty, shows an **email field only when the account lacks one** ("Finish
  your account"), and PUTs email + username together. `Account.email` is `@unique`, so a
  collision surfaces as "an account with this email already exists".
- On-device finding that reshaped this: **PostConfirmation fires for federated users**
  (the vault said it wouldn't) — the row exists by the time the tokens land; it's just
  incomplete. No Pre-Token handler needed.
- Apple email note: once the Cognito Apple IdP maps email → email, every Apple user has
  an email — Hide My Email yields a working relay address. The email field is mostly a
  safety net for phase-2 Facebook (accounts can genuinely lack one).

**Email, the final shape (2026-08-24, built):** the Apple IdP deliberately maps **no
email at all** — only the mandatory `username` (sub) mapping remains. With the email
mapping on, Hide-My-Email users landed a `@privaterelay.appleid.com` relay address, which
Skylar didn't want in the system; with no mapping, every social account arrives
email-less, the completion screen's email field always shows for them, and the app
collects the real address. `PUT /user` then pushes that email onto the consumer-pool user
via `AdminUpdateUserAttributes` with **`email_verified=true`** (`syncCognitoEmail` util in
nanza-api; IAM added in serverless.yml — needs a deploy). Two accepted trade-offs, both
Skylar's explicit call: (1) the address is marked verified without a code, so an address
can be squatted by someone who doesn't own it — revisit if account linking or email
recovery ever trusts the flag; (2) users who chose Hide My Email are asked for a real
email anyway, which App Review can frown on — worth a rethink before store submission.
Sync failures are log-and-continue: the DB email is the app's source of truth.

**The completion step's final form (2026-08-25, built): a LOCKED SHEET, not a screen.**
`components/modals/CompleteProfileModal` mounts on HomeScreen and gates purely on session
state — signed in, account loaded, no `profile.username` → the sheet is up, with no
backdrop tap, no swipe-down, no close. Home is visible behind it but untouchable. All
three completion paths share it: a fresh social sign-in, the email flow (ConfirmationCode
now clears the signup form, sets isSignedIn, and replaces to Main — landing on Home IS
the username step), and a relaunch that skipped completion — **which closes the known
hole below.** The `SignUpUsername` screen and its route are deleted; sign-up is two
screens (credentials, code) plus this sheet. Completion is implicit: the PUT lands,
`refreshCurrentUser()` re-derives the gate, the sheet unmounts itself.

**Email, superseding 2026-08-24:** the Apple email mapping is BACK ON (Skylar,
2026-08-25) — whatever Apple provides is saved, relay addresses included; the
hide-my-email objection was dropped along with the type-your-real-email friction and its
App Review risk. Social users therefore always arrive with an email and the sheet asks
username only. The sheet's conditional email field and nanza-api's `syncCognitoEmail`
(email + `email_verified=true` pushed to the pool) remain as the safety net for
providers that genuinely omit email — phase-2 Facebook.

~~**KNOWN HOLE (follow-up):** a social user who kills the app on the completion screen
re-enters with a live Cognito session and no username.~~ **CLOSED 2026-08-25** by the
locked sheet above — the gate is session-derived, so it simply reappears on every launch
until the username exists.

**Deferred (Skylar, 2026-08-24):** the ASWebAuthenticationSession consent prompt shows
the raw Cognito domain. A custom domain (`auth.nanza.app`, ACM cert + Route 53 alias)
would make it read "nanza.app"; backend token exchange would remove it entirely but is
the bigger job (custom-auth lambdas + API endpoint + per-provider token validation).
Living with the prompt in dev for now.

### Android: the username sheet under the keyboard (2026-08-26)

On a fresh social sign-in on Android the first keyboard open sat ON TOP of the locked
`CompleteProfileModal` instead of pushing it up. The shared `Modal` already pushes a
bottom sheet by the measured keyboard height (Reanimated `keyboardOffset`), so the sheet
itself wasn't the bug — two first-open edge cases in that push were:

- **A zero-height first `keyboardDidShow`.** Android under edge-to-edge can report
  `endCoordinates.height === 0` on the session's first show event. The listener now
  falls back to `Keyboard.metrics()?.height` and ignores a zero, and a sheet that mounts
  while the keyboard is already visible seeds its offset from `Keyboard.metrics()`.
- **The cap ran before the sheet was measured.** The Android push is capped so the
  sheet's top never crosses the safe-area top, using `sheetTopY` from `onLayout`; with
  `sheetTopY` still 0 the cap collapsed the push to 0. The cap now applies only once
  `sheetTopY > 0`.

Both are in `components/modals/Modal/index.tsx` and apply to every bottom sheet, not just
the username step.

## Open decisions

1. ~~**Where does `Login` live?**~~ **RESOLVED**: its own SHEET (`LoginModal`), opened
   straight from the header's Log In pill — no interstitial in front of it, since someone
   who already has an account has nothing to be sold. `AuthModal` is therefore sign-up
   only; its `initialStep` prop is now accepted-and-ignored so the two callers that still
   pass `"Login"` keep compiling.
2. ~~**Nav placement**~~ **RESOLVED**: root stack, alongside the other push-over-tabs
   screens.
3. **Copy** for the interstitial — placeholder shipped ("Join the community…"), still
   needs marketing.
4. **Abandonment**: what happens to a half-created Cognito user who quits at the code
   step? Already possible today; the screens make it more visible. Still open.
5. **Sparseness on device** — the accepted risk above. Worth a look before this ships.
