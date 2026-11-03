---
tags: [nanza-figma, handoff, nanza-mobile, messaging, sheets, tabs]
status: open
date: 2026-11-03
figma: https://www.figma.com/design/7c3Zz4nDUOutQG1BsnRRAw/Nanza?node-id=3186-62476&m=dev
fileKey: 7c3Zz4nDUOutQG1BsnRRAw
node: 3186:62476
nodes:
  - 3186:62476  # Inbox (MessagesScreen, All tab)
  - 3186:61007  # Thread (ConversationThread)
  - 3322:187866 # Reply menu (MessageActionSheet: quick reactions + edit/copy/delete)
  - 3186:60859  # Message action menu (Block user row)
  - 3322:188729 # Emoji menu (rn-emoji-keyboard)
  - 3186:60714  # Block user confirm
frame: ActivityScreen-All / default · Seller · edit me
repo: nanza-mobile
scope: styles + two new assets; logic items flagged for a later sprint
---

# Messaging — inbox, thread, sheets restyle

> [!warning] App-wide changes + things the dev team must add
> **These go beyond messaging:**
> - **Tab counter and tab text:** every `SegmentedTabs` and legacy `Tabs` strip, including the profile. The active-tab counter is now **dark, the same as inactive**. This reverses the light active chip in [[2026-11-02-profile-restyle]].
> - **Bottom/modal sheets:** every sheet gets the overlay, the white action button, and the 42pt float.
>
> **Assets the dev team adds to the project:**
> 1. **Balto Mobile Medium** font, for unread previews. Only `BaltoMobile-Book.ttf` ships today.
> 2. **Typing-indicator Lottie** JSON. `lottie-react-native` ^7.2.0 is already installed.
> 3. **Two new icons** for the reply menu: edit (pencil-in-square) and copy (two pages).
>
> **Logic, deferred:** date formatting and the reaction-count threshold. Confirm with the dev team before touching any logic; these can wait for a future sprint.

## Request (verbatim)
**Changed:**
- Update tab counter style (this applies to the profile tabs as well - tab styling should be universal)
- Update tab font styles (size, color, spacing from counter) - tab styling should be universal
- Unread messages take on new style with heavier Balto font (prompt dev to add to project)
- Typing indicator in the inbox screen and in threads shows Lottie animation (prompt dev to add to project)
- Update message cards in the inbox (match Figma 1:1), note that styling and date spacing is different for unread messages v. read messages
- Add overlay between bottom/modal sheets and page content so the modal is in focus
- Deprecate gray style buttons in bottom/modal sheets, replace with action buttons (white buttons used throughout the app, see 'Ready to join?' Modal
- Make bottom/modal sheet spacing from bottom of the viewport consistent with 'Ready to join?' Modal
- Reply menu for posts includes new SVGs (prompt dev to add to project)
- Style message reactions in our bottom/modal sheet style
- Date handling should update but confirm with dev before touching ANY logic – this can be done in a future sprint (only show year if over 1 year old, show date in abbreviated format, i.e Jan 1, and show relative date by time for today, yesterday for the day before, and the day of the week for anything within the past week) – message grouping (in threads) should follow the same pattern (rule and date stamp between posts)
- Emoji/reaction counter should only show if count is greater than 2 – confirm with dev, this can be done in a future sprint
- NOT IN MOCKS: Emoji category icons should use overlayText color (not black)

**Do not change:** anything else on these screens (assumed — no list given). Navigation, data and behaviour stay as they are except where a bullet says otherwise.

## Measurements
All frames are 402 wide ✓. Figma's variable fallbacks are light-theme values (white, #121212), so read the **token names**, not the hex.

### 1–2. Tabs (`SegmentedTabs`, Inbox 3186:62813)
| Element | Spec | Code today |
|---|---|---|
| Track | 4 padding, 1px `glassBorder`, `background` fill, radius max, 386 wide at x8 in the inbox | `segmentedTabsRow` ✓ |
| Segment | 34 tall, radius max; active fill `glassBorder` | 35 tall (`base + 15`) → **34** (owner: copy Figma) |
| Label | `pill` (SemiBold 13/14 +0.11). Inactive `secondaryText`, active `textSecondary` | `segmentedTabText` ✓ |
| Label → counter | 5 | `gap: xs - 1` ✓ |
| Counter | `small` (SemiBold 12/16), pad 6 sides / 2 bottom, radius max. **Dark chip on BOTH states** (owner, 2026-11-03): fill `glassBorder`, text `overlayText` | `countChip` ✓, but `countChipActive` / `countChipTextActive` still paint the light chip → **neutralise** |

### 3–5. Inbox row (`MessagesListRow`, 3186:62814)
| Element | Unread | Read |
|---|---|---|
| Row | pad 15 sides / 16 vertical, 1px `glassBorder` bottom rule | same |
| Avatar | 44 circle, 20 to the text column | same |
| Name | `title` (Bold 16/16), **`textSecondary`** | `title`, **`secondaryText`** |
| Unread dot | 8 circle, `action` fill, 6 after the name | none |
| Date | `label` (Medium 13/16 −0.11), `metaText`, flush right | same, but the name row pads **6 on the right**, so the date sits over the chevron |
| Name row → preview | 4 | 4 |
| Preview | **Balto Medium** 14/22 +0.11, `textSecondary`, 2 lines (44 tall), ellipsis | `body` (Balto Book 14/22), `metaText`, 2 lines |
| Trailing | **Count chip**, 24 after the preview, vertically centred: `bold` (SemiBold 14/16), `textSecondary`, pad 8 sides / 2 top / 4 bottom, radius max, fill `glassBorder` (owner: its own larger chip, not the tab counter) | `chevron-right` link icon 24 wide, top-aligned |
| Typing | — | the preview line becomes a chip: `surface` fill, 28 tall, pad 12 sides, radius max, three 7pt dots 4 apart (`metaText` / `overlayText` alternate = an animation frame). **Lottie replaces this** (asset 2) |

- The fuzzypitch row's date is Regular 14. This is a Figma artifact; use `label` like the other rows.

### 6–8. Sheets (all five overlay frames)
- **Overlay:** `background` @ 0.92 over the page (Figma `rgba(1,1,1,0.92)`). Code's `modalOverlay` already uses exactly this value. The fix is any sheet that **doesn't show it**: audit the `ActionModal` consumers (11 files), `ModalStack`, and rn-emoji-keyboard's `backdrop`.
- **Sheet:** glass fill + 6 blur, 1px `glassBorder` top and bottom, radius 44, **8 from each side (386 wide), 42 from the viewport bottom** (identical in all five frames). Content pad 20 sides / 36 vertical.
  - Code's `BottomSheet` float is `max(spacing.base, getBottomBarInset(insets.bottom))`, which gives 34 + 8 = 42 on the reference iPhone. That already matches.
  - The outliers are any sheet that doesn't use that float. The emoji picker today is "docked full width, shallow top corners" (`getEmojiPickerTheme`, 2026-11-02), and the new design floats it.
- **Action (white) button:** full width, 52 tall (18 vertical pad), radius 26, label `buttonLabel` (SemiBold 16/16) in `background`, fill `overlayText`. This is the AuthModal "Ready to join in?" `variant="action"` button. Replace gray (`muted` / gray `primary`) sheet buttons with it.

### 9. Reply menu (`MessageActionSheet`, 3322:187866)
- **Quick reactions:** 6 equal chips with a gap of 6. Each chip: `glassBorder` fill, radius 6, pad 12 vertical. Emoji at 20. The last chip holds the 20pt `emoji-add` icon.
- **Hint** "Double tap a message to ❤️": `thumbLabel` (Medium 12/14), `metaText`, centred, 20 below the chips.
- 10 between the header block and the rows.
- **Rows:** pad 16 vertical, 1px `glassBorder` rule between rows (none after the last). Each row has a 44 icon box holding a 20 icon, then 8 to the label. Labels are `buttonLabel` (SemiBold 16/16) in `text`.
  - **Edit:** new pencil-in-square SVG (node 3324:189602).
  - **Copy:** new two-pages SVG (node 3324:189591).
  - **Delete:** existing `trash-can`; label and icon use `error` (`#FF4D4F` = `system.error`).

### Block user (3186:60859 row, 3186:60714 confirm)
- **Row:** 44 close-icon box, then 8 to the text. Title `buttonLabel` in `text`, subtitle `subtitle` (Regular 14/16) in `metaText`, 4 apart.
- **Confirm:**
  - Title `large` (SemiBold 24/28 −0.11) in `text`.
  - Body `body` (Balto Book 14/22) in `metaText`, 8 below the title.
  - The white action button "Block" sits 14 below the body.

### 10. Reactions + thread (3186:61007)
- **Message:** pad 20 sides, 20 below each message.
  - Identity row: 31 avatar, 9 gap, name `title` in `textSecondary`, 3 gap, timestamp `thumbLabel` (Medium 12/14) in **`metaText`** (Figma uses a raw #BDC9D3; owner chose `metaText`), baseline-aligned.
  - Body: `postBody` (Balto 16/19 +0.25), `overlayText`, indented **40** (31 + 9), 4 below the identity row.
- **Reactions row:** 12 below the body, indented 40, chips 8 apart. Chip: `glassBorder` fill, radius 6, pad 9 sides / 5 vertical, emoji 18/18, count `micro` (SemiBold 10/14) in `overlayText`, 6 after the emoji. Add-reaction chip: pad 9 left / 7 right / 5 vertical, 18 `emoji-add` icon.
- **Date divider:** 1px `glassBorder` rule across the full width. Label `small` (SemiBold 12/16) in `inactive`, centred on a `background` patch with 12 padding each side, 20 below.
- **Emoji menu (3322:188729), same float as the other sheets:**
  - Search: glass, 44 tall, radius 22, placeholder `bodyInput` in `textPlaceholder`, 20 from the left.
  - Section label: `thumbLabel` in `metaText`.
  - Grid: 6 columns with a gap of 6. Cells 42 tall, `glassBorder` fill, radius 6, emoji 20.
  - Category bar: `glassBorder` fill, 34 tall, radius 8. 28 items with 22 icons. Selected item: 28 square, radius 6.
  - **Category icons: `overlayText`** (bullet 13).

### 11–12. Logic (deferred, confirm with the dev team first)
- **Dates (inbox + thread dividers):**

  | Message is from… | Show |
  |---|---|
  | today | the time ("5:23 pm") |
  | yesterday | "Yesterday" |
  | within the last week | the weekday ("Sunday") |
  | earlier this year | "Oct 21" |
  | more than a year ago | "Nov 2, 2025" |

  Thread groups take the same label on their divider rule.
- **Reaction count:** show the number only when the count is **2 or more** (owner, 2026-11-03; the notes said "greater than 2").

## Code map
Local `dev` may lag the team's branch. Wire each item to the matching component there.
- `src/styles/components/tabs.ts`
  - `countChip*`: THE counter, used by `segmentedTabCount*` and the legacy `countBadge`/`countText`. The active variants go.
  - `segmentedTab*`: the tab text. Consumers: every `SegmentedTabs` (Entity, Trade, MyTrades, CreateTrade, Messages, GroupMembers, Profile).
- `src/components/messages/MessagesListRow.tsx` + `src/styles/components/messages.ts`: the inbox row. Consumers: All / Active / Friends / SystemMessage tabs.
- `src/styles/components/typography.ts`: needs a **Balto Medium** slot (e.g. `bodyStrong`, 14/22 +0.11) once the font ships. Font props live only here.
- `src/hooks/useTypingIndicator.ts`: the typing data, unchanged. The thread renders "X is typing…" text today (`ConversationThread.tsx` ~l.221/428); the inbox has no typing UI locally.
- `src/components/modals/BottomSheet/index.tsx`, `Modal/index.tsx`, `ModalStack`, `ActionModal` (11 consumers), `styles/components/modals.ts` (`modalOverlay`, float comment ~l.404): the sheets.
- `src/components/global/ConfirmSheetContent.tsx` (`variant='primary'`), `ChoiceSheetContent.tsx` (`variant="muted"`): the gray buttons. AuthModal's `variant="action"` is the reference.
- `src/components/messages/MessageActionSheet.tsx`: the reply menu (`QUICK_REACTIONS`; icons `pencil`, `cards-2`, `trash-can`).
- `src/components/messages/ConversationThread.tsx` + `getEmojiPickerTheme` (`messages.ts` ~l.921): the thread, the reactions and the emoji picker.

## Open questions
Answered with the owner, 2026-11-03:
- **A.** Active tab counter: **dark chip on both states**, app-wide. This reverses the profile handoff's light active chip.
- **B.** Inbox row unread count: **its own larger chip** (`bold` 14, pad 8), not the tab counter.
- **C.** Thread timestamps: **`metaText`**.
- **D.** The reply menu is frame 3322:187866.

Answered with the owner, 2026-11-03 (second round):
1. **Balto Mobile Medium:** the dev team has the licensed file. They add it.
2. **Typing Lottie:** the dev team has the file. One chip in the inbox preview line; in the thread it replaces the "X is typing…" line.
3. **Reaction count:** show the number at **2 or more**. Hide it only for a lone reaction. (Logic, so the dev team confirms; can be a future sprint.)
4. **Timestamp placement:** "follow Figma". The date/time is **pushed to the right edge when there is an unread counter**, and sits **6 from the edge when read**. In the inbox this is the name row's 6 right padding on read rows. In the thread, the right-pushed time after "Today" is intentional, not an artifact. Follow the frame there; see Log.
5. **Thread header:** plain-text name only, **no avatar**. The avatar + name header in the overlay frames is the artifact.
6. **Emoji category bar:** keep the code's `containerActive` (`secondary.800`). Drop the separators if the library can't take a token.
7. **Segment height:** copy Figma, **34**. Compose it from tokens, since no single one matches.

## Prompt
```text
Restyle messaging and the shared sheets/tabs from the vault handoff
nanza-vault/nanza-figma/handoffs/2026-11-03-messaging-restyle.md: read it first, including
the Measurements tables and Open questions. Figma file 7c3Zz4nDUOutQG1BsnRRAw, frames
3186:62476 (Inbox), 3186:61007 (Thread), 3322:187866 (Reply menu), 3186:60859 (Block row),
3322:188729 (Emoji menu), 3186:60714 (Block confirm).

SCOPE
Styles, plus three assets the dev team adds (Balto Mobile Medium font, typing Lottie, two
reply-menu SVGs). No navigation or data changes. Items 11–12 are LOGIC: do not touch them
without the dev team's sign-off; they may be deferred to a later sprint. Change nothing else
on these screens. Don't port Figma artifacts: the fuzzypitch row's Regular-14 date, the raw
#BDC9D3 / #49505F / #FF4D4F, the white gradient fills on counters (light-theme fallbacks),
the avatar in the overlay frames' thread header (the header is plain-text name only).
Universal, no outliers: every change below edits the SHARED default (style class or
component), never a messaging-only variant. List each shared class's consumers for QA.

1. Update tab counter style (applies to profile tabs too, universal)
   tabs.ts countChip is THE counter. The owner chose a DARK chip on BOTH states
   (2026-11-03, reverses the 2026-11-02 light active chip): make countChipActive /
   countChipTextActive identical to the inactive look (glassBorder fill, overlayText text,
   `small` slot, pad 6 sides / 2 bottom, radius max), or delete their use if nothing else
   needs them. Covers segmentedTabCount* and the legacy countBadge/countText.
2. Update tab font styles (size, color, spacing from counter)
   `pill` 13/14, inactive secondaryText, active textSecondary, 5 to the counter. Code
   already matches; verify on every SegmentedTabs consumer. Segment height 35 → 34 (copy
   Figma; compose tokens, no raw number).
3. Unread messages, heavier Balto
   Add the Balto Mobile Medium font file (asset 1, the dev team has it; iOS + Android
   linking), then ONE new typography slot in typography.ts (e.g. bodyStrong: Balto Medium
   14/22 +0.11). Unread preview uses it in textSecondary.
4. Typing indicator, Lottie (inbox + thread)
   Add the Lottie JSON (asset 2, the dev team has it) with lottie-react-native (installed).
   Inbox: on a typing conversation the preview line becomes the chip (surface fill, 28
   tall, pad 12 sides, radius max) holding the animation. Thread: the same chip replaces
   the "X is typing…" text. Typing data comes from useTypingIndicator; wire the inbox to it only if the
   team's branch exposes typing per conversation, otherwise flag it.
5. Message cards in the inbox (match Figma 1:1; unread ≠ read)
   MessagesListRow per the handoff's Inbox table: pad 15/16, 44 avatar, 20 gap, name
   `title` (unread textSecondary / read secondaryText), 8pt `action` dot 6 after an unread
   name, date `label` metaText (read rows pad the name row 6 on the right so the date
   sits over the chevron), 4 to the preview, preview 2 lines (unread bodyStrong
   textSecondary / read `body` metaText), trailing 24 after: unread = count chip (`bold`
   textSecondary, pad 8 sides / 2 top / 4 bottom, radius max, glassBorder fill), read =
   chevron. glassBorder bottom rule.
6. Overlay between sheets and page content
   modalOverlay (background @ 0.92) is already the Figma value. Find every sheet that
   doesn't show it (ActionModal consumers, ModalStack, rn-emoji-keyboard backdrop,
   overlayMaxOpacity overrides) and route it through the shared overlay.
7. Deprecate gray sheet buttons → white action buttons ("Ready to join?" reference)
   ConfirmSheetContent / ChoiceSheetContent and any sheet using muted / gray primary:
   use AuthModal's variant="action" (52 tall, radius 26, buttonLabel in background,
   overlayText fill). Change the shared components' defaults; list consumers.
8. Sheet spacing from the viewport bottom, consistent with "Ready to join?"
   Every sheet floats 8 from each side and 42 from the bottom on the reference device:
   BottomSheet's max(spacing.base, getBottomBarInset) rule. Move outliers (the docked
   full-width emoji picker, anything else not using that rule) onto it. Radius 44,
   glassBorder top/bottom, content pad 20 sides / 36 vertical.
9. Reply menu, new SVGs
   Export the edit (3324:189602) and copy (3324:189591) icons from Figma into the Icon set
   (asset 3); replace `pencil` / `cards-2` in MessageActionSheet. Spec in the handoff:
   6 equal reaction chips gap 6, hint `thumbLabel` metaText 20 below, rows pad 16 with
   glassBorder rules, 44 icon box + 20 icon + 8 gap, `buttonLabel` text; Delete in
   theme.colors.error (label and icon). Block row and Block confirm per the handoff.
10. Message reactions in the bottom/modal sheet style
   Reaction chips (glassBorder, radius 6, pad 9/5, emoji 18, count `micro` overlayText 6
   after), add-reaction chip, the emoji picker restyled as a floating glass sheet (search
   44 / radius 22, 6-col grid gap 6, cells 42 radius 6, category bar 34 radius 8), and
   thread message layout (identity row, 40 body indent, timestamp `thumbLabel` metaText,
   date divider `small` inactive) per the handoff. Timestamps follow Figma: pushed to the
   right edge in the unread state, inline otherwise (see the handoff's Q4 / Log). Thread
   header: plain-text name only, no avatar.
11. Date handling (LOGIC, confirm with the dev team, may defer)
   today → time · yesterday → "Yesterday" · within 7 days → weekday · else "Jan 1" · over a
   year → "Nov 2, 2025". Thread dividers follow the same labels. Don't implement without
   sign-off.
12. Reaction counter only when count is 2 or more (LOGIC, confirm, may defer)
13. Emoji category icons use overlayText (not black)
   getEmojiPickerTheme category.icon / iconActive → overlayText-based (theme fallback
   chain, no hex).

COUPLED / TRAPS
- countChipActive was set on 2026-11-02 for the profile; changing it changes the profile
  tabs too. That's intended.
- getEmojiPickerTheme's comment records a 2026-11-02 decision to DOCK the picker; this
  handoff supersedes it. Update the comment.
- Sheet float math lives in BottomSheet and Modal (inline) and is documented in modals.ts.
  Change the rule in one place only.
- New font: Android needs it in android/app/src/main/assets/fonts; iOS in Info.plist
  UIAppFonts. Run pod install / a clean build.

HOUSE RULES
Plan first with confidence scores (≥3, alternatives under 0.8). Escalate anything
touching src/styles/utils/* or src/contexts/*. Style buckets only, no inline styles,
typography slots only (font props only in typography.ts), theme tokens only with
theme-based fallbacks, no raw sizes; compose tokens where none match. Proportional sizing
off the 402 frame. Lint src (not npm run lint), tsc, then /ui-review. Don't commit.
```

## Log
- 2026-11-03 — handoff written (6 frames). Owner answers A–D recorded. Supersedes [[2026-11-02-profile-restyle]] on the active tab counter colour.
- 2026-11-03 — second round answered (Q1–7): dev team has the font + Lottie; reaction count shows at 2+; thread header plain text, no avatar; segment 34; emoji bar keeps `containerActive`. Timestamp rule: right edge when unread, 6 from the edge when read. In the thread, the "unread" trigger for the right-pushed time is not yet pinned down; the dev team matches the frame and confirms the condition with the owner.
