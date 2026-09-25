---
tags: [nanza-mobile, solution-design, seller, onboarding]
---

# Seller Onboarding

> **Status: living design.** The three-step "Get ready to sell" flow (identity → payout method →
> shipping preferences) that gates seller access, hosted on the Account tab's Selling screen
> until `hasSellerAccess` flips.

## Shape

- **Hub** — `SellerOnboardingScreen`, rendered inside `AccountScreen/Selling` while the viewer
  has no seller access (latched at mount so a mid-flow Stripe webhook can't swap the screen
  out from under them). A `StepIndicator` lists the three steps in order; each opens its own
  screen. "Start selling" is the hub's single `DetailActionBar` CTA: it re-checks Stripe's
  outstanding requirements, waits a beat for the webhook, refreshes the session and closes.
- **Confirm your identity** — `ConfirmIdentityScreen`: two in-screen steps (personal info →
  address) driving one `DetailActionBar` (Continue → Submit). Submit creates or updates the
  Stripe-backed seller; success routes on to ID capture when no document is on file.
- **ID capture → confirm → review** — `IDCaptureScreen` (vision-camera, the scan screen's
  shutter recipe), `IDConfirmScreen` ("Looks good" / "Retake"), `IDReviewScreen`.
- **Payout method** — `PaymentMethodScreen` (Stripe `CardField`, debit only).
- **Shipping preferences** — `ShippingPreferencesScreen` (the Selling screen's two panels,
  reused).

## Presentation rules (2026-09-11)

- **Every screen in the flow is a vertical sheet with the glass down-chevron.** All of them
  use `PageLayout`'s floating header with `backIcon: 'chevron-down-2'` — the same treatment
  as PostDetail, the offer screens and the bulk flow. The left arrow (`chevron-left-3`) that
  the identity/payout/shipping screens shipped with is gone; `IDCaptureScreen`, which drew an
  in-flow `Header` (smaller title, left arrow), moved onto the same floating header so its
  title matches the flow.
- **The ID screens inherit the stack's default vertical transition.** Their AppNavigator
  registrations used to override to `forHorizontalIOS` with a horizontal gesture; those
  overrides were removed (Skylar approved the navigator edit by voice) so capture, confirm
  and review enter from the bottom like every other onboarding screen.
- **Busy CTAs spin.** Every bottom-bar primary in the flow passes its in-flight state as
  `loading` (the Button's Lottie arc, label kept): identity Submit (`isSaving`), Add card
  (`isLoading`), shipping Update (either panel's save state), Looks good (`isSubmitting`),
  Start selling (`isStartingToSell`). Continue on the identity step is a synchronous step
  switch and has nothing to wait on. The reason none of them *appeared* to spin until
  2026-09-11 was the asset, not the wiring — see [[design-system|Design System]] § Button's
  loading arc.
- **Confirm your identity keeps its fields above the keyboard.** The screen's `PageLayout`
  scrolls with `automaticallyAdjustKeyboardInsets` (iOS insets + the Android measured pad),
  the two step views are plain Views inside it (they used to be their own ScrollViews under a
  non-scrolling page, which is why the DOB row and SSN field hid under the keypad), and the
  action bar no longer sticks to the keyboard — the CreateGroupScreen / Edit-profile rule,
  since a bar riding above the keys stacks over the focused field.
- **Payout method matches the identity form's frame (2026-09-11).** The screen is titled
  "Add payout method" (the hub's step label; it read "Add payment method"), its intro sits in
  the identity step's `pageTitleLgContainer` + `confirmIdentityHeaderContainer` wrapper and the
  Stripe `CardField` in a `formStack`, so both onboarding forms share one left edge.
- **Shipping preferences shares the frame too (2026-09-11).** Intro in the identity step's
  wrapper, each section in a `panelFlat` (the Seller preferences screen's own treatment of
  `ShippingOptions` / `TrackedPackages`), headings phrased to the seller — "Your shipping
  options" and "Do you want to support tracked packages?" — and `TrackedPackages`' intro line
  ("Select tracked package options below…") removed everywhere it renders; the heading carries
  it.
