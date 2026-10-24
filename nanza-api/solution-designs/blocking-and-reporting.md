---
tags: [nanza-api, nanza-mobile, nanza-admin, solution-design, plan, moderation, connections, posts]
---

# Blocking Visibility & Content Reporting — Plan

> **Status: BUILT (all 5 steps), 2026-08-23 (Skylar, voice session).** Uncommitted;
> the Report migration is Skylar's to run. Apple App Review release
> blockers. Two independent asks that share one surface (the post action row) and one
> theme (user-controlled moderation): **(1)** blocking must hide content, not just
> messaging; **(2)** a reporting model + flow + admin queue.
>
> Extends [[open-posting-and-friend-visibility|Open Home Posting & Friend-Scoped
> Visibility]] — this resolves that plan's **open decision #2** ("should a BLOCKED
> connection also hide the affiliate posts of the blocked party"): **no** — affiliates
> are out of scope for blocking (see Decisions).

## The ask (Apple)

Apple flagged two requirements for release:

1. **Blocking must actually hide content.** If I block a user, I stop seeing their
   posts and replies — in the home feed *and* inside groups. In a group we both stay
   participants; I just don't see them.
2. **Reportable content.** Posts need a report affordance with a reason taxonomy
   (the X/Twitter shape), and Nanza needs somewhere to review reports.

## Today's wiring (mapped 2026-08-23)

**Blocking exists but is messaging/discovery-scoped only.** `ConnectionStatus`
(schema.prisma:234) is `PENDING | ACCEPTED | BLOCKED`; `Connection` (schema.prisma:1703)
is the symmetric pair row with `@@unique([initiatorId, recipientId])` and indexes on
both sides. `BLOCKED` is enforced in:

- `routers/search.ts:126-138` — blocked accounts excluded from people search.
- `routers/message.ts:70,105` + `routers/conversation.ts:45` — no DMs.
- `routers/order.ts:1334`, `routers/refund.ts:141`, `routers/group.ts:935`,
  `routers/inboxCount.ts:24` — assorted relationship gates.
- `validation/connection.ts:18,29-33` — `assertNotBlockedBetween`-style guard.

**It is NOT enforced anywhere in the post read path.** The brand feed's filter is
`brandVisibilityArms` (validation/post.ts:189-197):

```ts
OR: [
  { account: { profile: { type: 'AFFILIATE' } } },        // ← unconditional
  ...(viewerAccountId ? [{ accountId: { in: [viewer, ...friends] } }] : []),
]
```

Two consequences, both confirmed by reading the filter:

- **Blocking a BASIC user already hides their brand posts** — but incidentally, not by
  design: blocking replaces/precludes `ACCEPTED`, so they drop out of the friend arm.
  Skylar's assumption ("I think that already works that way") is *correct for basic
  users only*.
- **Blocking an AFFILIATE hides nothing** — arm 1 admits every affiliate post
  regardless of connection state. This is the real gap in the home feed.

**Groups do no per-viewer filtering at all.** `GET /posts` with `postableType=GROUP`
gates on `isActiveGroupParticipant` (routers/post.ts:~280) and then returns every
post in the group to every member. `isRootPostViewable` (validation/post.ts:218-245)
likewise checks group membership only.

**No Report model exists** — `grep 'model Report' prisma/schema.prisma` is empty.
Greenfield.

**Mobile** already has block UX (`ActionModal/Feed.tsx:70`, `useMessageUser.tsx`,
`ConnectionButton`), and `assets/icons/warning.svg` already ships. The post footer
action row is `components/posts/PostItem.tsx:406-435` (reply, heart, share).

**Admin** (`src/app/`) has no Posts section — Products, Orders, Sellers, Tags, etc.
only. Routes are a flat table in `src/app/AppLayout.tsx:29-41`.

## Decisions (Skylar, 2026-08-23 voice)

1. **Blocking is symmetric** — "it goes both ways, we don't see each other." One
   `BLOCKED` row hides content in *both* directions, matching how messaging already
   treats it. No initiator/recipient asymmetry.
2. **Affiliates are out of scope for blocking** — "you don't have a connection with
   affiliates… being able to report an affiliate should hopefully be okay."
   This **closes open decision #2** of the friend-visibility plan. Practically the
   affiliate arm stays unconditional, and reporting is the lever against affiliate
   content. *(Note: this means the home-feed block gap above is mostly theoretical
   today — blocking a basic user already works. The group gap is the real work.)*
3. **Hide blocked content entirely — no placeholder.** "Just remove them, we can hide
   them altogether, their posts, replies, the whole thing."
4. **Don't deep-filter replies inside an existing thread** — "we don't have to remove
   their replies from an existing conversation, not that deep." Root-level filtering
   is enough; a visible root brings its thread, consistent with the existing
   "replies are conversation, not feed placement" posture.
5. **Reports are one-to-many on Post** — a post accumulates many reports, each with
   exactly **one** reason plus optional free text (radio buttons, per the X
   screenshot Skylar sent and confirmed on the call).

## Design — Part 1: blocking hides content

**No schema change.** Like friend-visibility, this derives from live state: unblock
someone and their content returns on the next read. Same posture as the affiliate
demotion / unfriend behavior already documented.

### The batch helper

`services/connection.ts` gains the block-shaped sibling of
`findAcceptedConnectionAccountIds`:

```ts
// Symmetric: one BLOCKED row hides content both ways, so the OR covers
// either ordering and maps to the other party — the same shape as the
// accepted-connection lookup beside it.
export const findBlockedConnectionAccountIds = async (accountId: string): Promise<string[]>
```

One `findMany` on `status: 'BLOCKED'`, `OR: [{initiatorId}, {recipientId}]`, mapped to
the other party. Fetched **once per request**, like the friend ids.

### The filter

A single exported builder in `validation/post.ts`, beside the existing ones (the
"one home for the rule" posture that file already keeps):

```ts
// Blocked accounts' content is hidden symmetrically, in every home. Unlike
// brandVisibilityArms this is a NOT arm — it subtracts rather than admits, so
// it ANDs onto whatever visibility the home already computes.
export const notBlockedFilter = (blockedAccountIds: string[]): Prisma.PostWhereInput =>
  blockedAccountIds.length ? { accountId: { notIn: blockedAccountIds } } : {}
```

Applied by ANDing onto the `where` of every **root** read that can surface another
account's post:

- `GET /posts` **home feed** (BRAND) — alongside `brandPostVisibilityFilter`.
- `GET /posts` **group feed** (GROUP) — the new coverage; membership gate unchanged,
  so both parties stay in the group and only the rendering is filtered.
- `GET /posts` **author feed** (`accountId` mode) — a blocked author's profile Posts
  tab reads empty.
- The **tag-page union** (`taggedTagValuePostsFilter`) and **item-chat** arm
  (`referencedPostsFilter`) — both already take `(viewer, friendAccountIds)`; they
  grow a `blockedAccountIds` parameter so blocked content can't re-enter through a
  side surface. This mirrors exactly how the no-leak rule was threaded through them.
- `isRootPostViewable` — a blocked author's root 404s for the blocker, so detail
  reads and share codes can't bypass the feed filter.

Per decision #4, thread reads (`parentId` set) do **not** apply it.

### Counts stay consistent

`paginatePrisma`'s parallel count shares the `where`, so `total` (mobile's band
overflow / See-all driver) stays correct for free — the same property the
friend-visibility work relied on.

## Design — Part 2: reporting

### Schema (nanza-api)

> Per repo rules: **no comments in schema.prisma**, and **Skylar runs migrations
> himself** — this plan specifies the model; it does not add migration files.

Reasons and their labels are taken verbatim from the X screenshot Skylar sent
(2026-08-23), in the same order it lists them:

| Enum | Label |
| --- | --- |
| `SPAM` | Spam |
| `HATE_ABUSE_OR_HARASSMENT` | Hate, Abuse, or Harassment |
| `CHILD_SAFETY` | Child Safety |
| `VIOLENT_SPEECH` | Violent Speech |
| `GRAPHIC_OR_VIOLENT_MEDIA` | Graphic or Violent Media |
| `ILLEGAL_OR_REGULATED_BEHAVIORS` | Illegal and Regulated Behaviors |
| `IMPERSONATION` | Impersonation |
| `ADULT_SEXUAL_CONTENT` | Adult Sexual Content |
| `PRIVATE_OR_NON_CONSENSUAL_CONTENT` | Private or Non-Consensual Content |
| `SUICIDE_OR_SELF_HARM` | Suicide or Self-Harm |

```prisma
enum ReportReason {
  SPAM
  HATE_ABUSE_OR_HARASSMENT
  CHILD_SAFETY
  VIOLENT_SPEECH
  GRAPHIC_OR_VIOLENT_MEDIA
  ILLEGAL_OR_REGULATED_BEHAVIORS
  IMPERSONATION
  ADULT_SEXUAL_CONTENT
  PRIVATE_OR_NON_CONSENSUAL_CONTENT
  SUICIDE_OR_SELF_HARM
}

enum ReportStatus {
  PENDING
  REVIEWED
  ACTIONED
  DISMISSED
}

model Report {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  reason ReportReason
  detail String?      @db.VarChar(1000)
  status  ReportStatus @default(PENDING)

  post   Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId String

  reporter   Account @relation(fields: [reporterId], references: [id], onDelete: Cascade)
  reporterId String

  reviewedAt DateTime?
  reviewNote String?   @db.VarChar(1000)

  @@unique([postId, reporterId])
  @@index([postId])
  @@index([status, createdAt])
}
```

Notes on the shape:

- `reason` is a **single** value, not a list — the X screenshot Skylar sent uses radio
  buttons, and he confirmed the same (2026-08-23): "it's not checkboxes, but it's radio…
  just being able to click between different ones."
- `@@unique([postId, reporterId])` makes re-reporting idempotent (update, not
  duplicate) and stops one account inflating a post's count.
- `@@index([status, createdAt])` is the admin queue's read path.
- `Post` gains `reports Report[]`; `Account` gains the reporter back-relation.

### Endpoints

- `POST /post/:id/report` — body `{ reasons, detail? }`. Auth required; upsert on
  `(postId, reporterId)`. Returns 404 through `isRootPostViewable` if the reporter
  can't see the post in the first place.
- `GET /reports` (admin) — filter by `status`, paginated via `paginatePrisma`,
  including post + author + reporter for the queue.
- `PATCH /report/:id` (admin) — set `status`, `reviewNote`, `reviewedAt`.

### Mobile flow

Per Skylar's description:

1. **Flag icon** appended to the end of the `PostItem` footer action row
   (PostItem.tsx:406-435, after reply / heart / share). Uses the existing
   `assets/icons/warning.svg` for now; swap for a real flag glyph when available.
2. Tapping opens a **screen** (not a sheet) with the standard back arrow, header
   copy **"What are you reporting?"**.
3. **Radio list** (single select) of the reason enum values with the labels above,
   plus a free-text field for detail. A **Cancel** action sits top-right and a
   **Next / Submit** button pins to the bottom, disabled until a reason is picked —
   matching the reference screenshot.
4. Submit → `POST /post/:id/report` → confirmation, back to the feed.

### Admin flow (nanza-admin)

New **Posts** section — `src/app/Posts/`, route `/posts` added to the flat table in
`AppLayout.tsx`, and a nav entry beside Orders/Sellers.

- Lists **all** posts (Skylar: "we can see posts… and then just be able to filter
  those posts by if it has a report or not on it").
- A **filter dropdown**: All posts / Reported only / by `ReportStatus`.
- Row detail shows the post, its author, and its accumulated reports with reasons,
  detail text, and reporter — with the `PATCH` actions to resolve them.

## Implementation record (2026-08-23, uncommitted)

**Steps 1–4 built same session.** Specifics that differ from, or sharpen, the design:

- `services/connection.ts`: `findBlockedConnectionAccountIds(accountId)` — one findMany
  (`status BLOCKED`, bidirectional OR), mapped to the other party. Mirrors its accepted
  sibling exactly.
- `validation/post.ts`: `notBlockedFilter` added; `taggedTagValuePostsFilter` and
  `referencedPostsFilter` grew a fourth `blockedAccountIds` param (defaulted `[]`, so
  existing call sites stay valid). `isRootPostViewable` gained a block check that runs
  **before** the home rules and after the DRAFT check — self-reads short-circuit under it.
- `routers/post.ts`: `blockedAccountIds` fetched once per request, gated on
  `viewerAccountId && !parentId` — i.e. **any root read, every home**, not just BRAND
  (that is what closes the group gap). ANDed onto all four `where` branches. The author
  feed returns an early empty page (`{data: [], page, total: 0}`) when the viewer and
  author are blocked, rather than filtering — the whole tab goes dark, not just its BRAND rows.
- Schema: `Report` + `ReportReason` (10 values) + `ReportStatus` (4 values), `Post.reports`,
  `Account.reportsFiled`. `prisma validate` clean; **migration NOT run — Skylar's to run.**
- `routers/report.ts` (new, mounted in `all_routes.ts`): `POST /post/:id/report` (upsert on
  the `(postId, reporterId)` unique, so re-reporting rewrites and resets to PENDING;
  refuses self-reports; vets visibility through `isRootPostViewable` on the ROOT id so a
  reply is only as reportable as its thread), `GET /reports` (admin, status/postId filters),
  `PATCH /report/:id` (admin; PENDING clears `reviewedAt`/`reviewNote`).
- **Mobile** (step 4): `services/api/Report.ts` (service + `REPORT_REASON_OPTIONS` carrying
  the display copy), `screens/posts/ReportPostScreen/` (reuses the existing `RadioField`,
  `TextArea`, `Button`, `PageLayout` — no new primitives), `styles/components/report.ts`,
  a `ReportPost` route param + AppNavigator registration (**P0 nav-edit escalated and
  approved by Skylar on the call** — "we use screens for all that stuff"), and the flag in
  `PostItem`'s footer row (hidden on your own post; guests get the auth modal, matching
  the heart/reply posture).
- **Flag icon + guest gate (2026-08-23, second pass):** the real `flag.svg` replaced the
  `warning` placeholder (registered as `'flag'` in `iconMap.ts`; `warning` stays for other
  callers). The asset shipped with a hardcoded `fill="black"`, which a path-level fill
  would keep regardless of the Icon's `color` prop — swapped to `fill="currentColor"` so
  it takes the footer's muted tint like its neighbours. Also: the flag now renders for
  **signed-in viewers only** (Skylar) — unlike the heart and reply, which show for guests
  and open the sign-up window, reporting is simply absent for them, so `handleReportPress`
  no longer carries a guest branch.
- **Report-sheet redesign (2026-08-23, third pass — Skylar on the call):**
  title moved OUT of the floating header into the page body (`typography.large` +
  `body` subtitle, the modals.ts sheet-title pair); reason labels bolder/bigger with
  the radio moved to the RIGHT — both added to the shared `RadioField` as opt-in
  `prominentLabel` / `trailingControl` props rather than forking it, since cart and
  shipping options depend on its current layout — `prominentLabel` settled on
  `typography.title` (Bold 16) after Skylar asked for the labels "a bit bigger"; submit is now `variant="action"`
  `size="xxl"` (the buy/sell CTA — white fill, ink label) and **pinned** as a sibling
  AFTER `</PageLayout>`, the shape PostDetail uses for its reply composer. A first
  attempt nested a ScrollView inside PageLayout's own and the footer rode off-screen:
  `pageScrollContent` only `flexGrow`s, so the inner scroller was never height-bounded.
- **Reuse `DetailActionBar` (2026-08-23, fourth pass):** the hand-rolled pinned footer
  was replaced outright by the listing/bid CTA bar. A plain opaque footer left content
  visibly colliding with the button as it scrolled past; `DetailActionBar` is an
  absolute overlay whose gradient backdrop **fades content out beneath it**, and it
  already carries safe-area insets, `stickToKeyboard`, and the `variant="action"`
  `size="xxl"` button with `inactive` ghosting — i.e. everything that had been
  reimplemented by hand. (`stickToKeyboard` was dropped with the OTHER field: nothing
  on the sheet takes text input any more.) Clearance now comes from `useActionBarPadding()`, derived from
  the same `getActionBarGeometry` the bar paints with, so the two cannot drift (the
  hardcoded `spacing['75']` is gone). `report.ts` lost its `footer` style entirely.
- **`OTHER` reason — added then PULLED the same day (2026-08-23).** Briefly added to the
  enum with a conditional, required free-text field. Skylar pulled it: "we don't need
  support for other right now." Reverted in all three repos — the enum is back to **ten**
  values and no client collects free text.
  **The `Report.detail` column and its API validation deliberately REMAIN**, so
  re-introducing the flow is a client-side change with no migration. Mobile's
  `CreateReportPayload` no longer carries `detail`; admin still RENDERS `report.detail`
  when present, since a historical report could hold one.
  *(Interlude worth keeping: while OTHER existed, the DTO-derived `ReportReason` caught
  the missing enum value at compile time — the drift protection working as intended.)*
- **Radios deselect** — tapping the chosen reason clears it, returning the sheet to its
  opening state, so a mis-tap doesn't strand you on a choice you must submit or back out of.
- **flag.svg fills:** the asset shipped with `fill="black"` on the path AND `fill="none"`
  on the root; either one beats the `fill` prop `Icon` forwards, so the glyph rendered
  off-colour next to its neighbours. Both attributes were REMOVED — `share-3` and
  `heart-2`, the working footer glyphs, carry no fill of their own either. An
  intermediate `currentColor` attempt did NOT work: nothing forwards a `color` prop to
  the SVG root.
- **Share affordance now matches the mint gate (2026-08-23).** Skylar hit
  *"Posts visible to friends only cannot be shared by link"* from the share icon: the
  client offered an action the API refuses. `POST /post/:id/share` rejects three shapes —
  replies, GROUP-homed posts, and **BRAND-homed posts whose author is not an AFFILIATE**
  (friends-only; the public resolver has no viewer to friend-check). The client could not
  evaluate the third: `postInclude` sent `profile` as id/username/avatar only.
  - **API:** `profile.type` added to `postInclude` (so every post read, feeds included via
    `postListInclude`, carries it).
  - **Mobile:** new `useCanSharePost(post)` beside `useCanShare` — the viewer gate
    (`useCanShare`) AND the post gate (the three server rules) in one place, consumed by
    BOTH share entry points (`PostItem`'s footer, `PostDetailScreen`'s header circle).
    `useCanShare` stays as-is for entities/groups/tags/profiles.
  - Note the rule is **narrower than "non-affiliates can't share"**: a basic user's post
    in a listing/bid/entity/tag chat still shares fine — only the BRAND home is
    friends-only. The hook documents this so it isn't over-applied later.
  - **Shipped broken, then fixed the same session.** The first version compared
    `type === 'AFFILIATE'` strictly, so a missing field read as "not an affiliate" and
    hid the share icon on EVERY post. A type probe had confirmed `profile.type` is
    typed on `HydratedPostDto` — but the type existing says nothing about the RUNTIME
    payload: the app points at `https://api-dev.nanza.app` (mobile `.env`), and the
    `postInclude` change is local and **undeployed**, so the live response still carries
    only `{id, username, avatar}`. Confirmed by querying the dev API directly rather
    than reasoning about it.
    The hook now **fails OPEN**: unknown author type → show the icon and let the API be
    the authority (the 400 is already handled). Hiding a working affordance is the worse
    failure. This also means the client needs no coordinated deploy — it degrades to
    today's behavior until the API ships, then tightens automatically.
  - **Lesson:** a compile-time probe verifies the type, not the data. For a field newly
    added to a server include, check the actual response from the environment the client
    talks to.
  - This mirrors a server rule on the client by necessity — the affordance must be
    decided before the request. If the mint gate changes, `useCanSharePost` must follow.
- **Duplicate reports: refused, not tracked (2026-08-23).** A `viewerHasReported`
  flag was built first (a `withPostReports` helper mirroring `postLikes`, plus a
  "Report sent" label swapping out the flag, plus query invalidation). Skylar: *"I don't
  like how complex it got"* — and chose the cheaper shape: **the app tracks nothing; the
  tap finds out.**
  - `POST /post/:id/report` no longer upserts. It looks for an existing
    `(postId, reporterId)` row and returns **409** with the user-facing copy
    *"You already reported this post. Our team is looking into the matter."*, with a
    `P2002` catch covering two submits racing past the pre-check. 409 not 400: the
    request is well-formed, it conflicts.
  - The client renders that message as-is — `fetchData` already throws with the API's
    `errorMessage`, so the API's copy IS the UI string. No client-side state, no extra
    query per feed page, no invalidation. `utils/postReports.ts` deleted.
  - **Trade-off accepted:** the flag looks identical whether or not you've reported, so
    a repeat reporter picks a reason before learning it was already filed. That's the
    price of not shipping per-viewer report state on every post.
  - Side effect worth keeping: the screen's LOCAL `AlertProvider` was removed. `alert`
    uses a single save/restore listener slot, so a provider that unmounts alongside
    `goBack()` would swallow its own alert — `AppNavigator`'s root provider (which wraps
    everything) is the correct owner, and the 409 message lands on the feed behind.
- **Admin post view shows content items (2026-08-23, Skylar):** *"just so we can ensure
  content items are safe"* — text alone can't moderate a post whose payload is an image
  or a link. **No API change needed**: the admin read already uses `postInclude` (an
  `include`, not a narrowing `select`) and skips `trimPostsForList`, so full items —
  `media`, `url`, `body`, `referenceId`, and nested `children` — were already on the
  wire and simply unrendered.
  - New `app/Posts/PostContentItems.tsx`: one card per item with its type/primary/index
    badges. **Images render inline** (that's the point), links render as `target="_blank"`
    + `rel="noreferrer noopener"` — the URL is user-submitted and must not get a handle
    on the admin window — and reference items show the id they point at rather than
    resolving it. GROUP items are galleries, so the component recurses one level into
    `children`.
  - The modal's old text flattening (which concatenated every item `body` into loose
    lines) is superseded and removed; the post block now shows only `post.body`, with
    everything else in the new Content section.
  - Images use `REACT_APP_S3_IMAGE_BASE_URL` directly (full size, no resizer) — right for
    moderation, and the existing admin convention. Note this var is injected by the
    deploy workflows and is **not** in the local `.env`, so images resolve in deployed
    builds only — pre-existing for every image in the admin app, not new here.
- **Known cosmetic deviation:** the shared `RadioField` draws its circle on the LEFT; the
  X reference has it on the right. Kept the shared component rather than forking it.
- Verified: API `tsc --noEmit` + eslint clean. Mobile `tsc` introduces **zero** new errors
  (252 pre-existing repo-wide, unchanged) and eslint clean on all touched files.

**Step 5 (admin) also built.** It needed an API addition the plan hadn't anticipated:
`GET /posts` had **no admin read** — it required either `postableType`+`postableId` or
`accountId`, so there was no "every post" query for a dashboard. Added an `adminId` mode
(admin-gated, roots only, no viewer scoping — moderation must see what the feeds hide),
with `reported=true` and an optional `reportStatus` to narrow. It deliberately uses
`postInclude` rather than `postListInclude`: the list shape drops content-item `body`,
and moderation has to read the text it's judging. `Account.email` is selected only on
this read, as the queue's fallback label when a profile has no username.

- nanza-admin: `services/api/Post.ts` + `services/api/Report.ts`, `app/Posts/`
  (index + `PostReportsModal` + `data/fetchPosts.ts` + `types.ts`), a `/posts` route and
  a TopNavbar entry. Filter dropdown is **Reported only (default) / All posts**, with a
  second status dropdown inside the reported view. Resolve actions are
  Action / Dismiss / Reopen per report.
- **Types resolved (`@oakplatforms/types@0.1.95`, 2026-08-23):** the package now ships
  `ReportDto` and `PostDto.reports`, so the temporary `app/Posts/types.ts` extension was
  **deleted**. Both apps now derive `ReportReason`/`ReportStatus` from the DTO
  (`NonNullable<ReportDto['reason']>`) rather than hand-kept unions — verified identical
  to the API enum at the swap, and a future enum change now surfaces as a type error.
  `PostDto`/`ReportDto` re-exported from each app's `types/index.ts`. Note the generated
  DTO makes every field optional, so the admin modal guards `report.id` before PATCHing
  instead of asserting.
- Verified: admin `tsc --noEmit` clean, eslint clean on all touched files (one
  pre-existing unused-`catch` error in TopNavbar.tsx:45 is untouched and unrelated).

## Rollout

1. **API — blocking.** `findBlockedConnectionAccountIds`; `notBlockedFilter`; thread
   it through the brand feed, group feed, author feed, tag union, item-chat arm, and
   `isRootPostViewable`. No migration.
2. **API — reporting schema.** `Report` model + two enums + back-relations.
   *Skylar runs the migration.*
3. **API — reporting endpoints.** `POST /post/:id/report`, `GET /reports`,
   `PATCH /report/:id`.
4. **Mobile.** Flag icon in the post footer → report screen → submit. Verify blocked
   content is gone from home, groups, profile tabs, and tag pages.
5. **Admin.** Posts section + reported filter + resolve actions.
6. **Fold down.** Durable outcome into [[posts|Posts]] (a visibility table row for
   blocking) and a new admin solution design; retire this plan; close open decision
   \#2 in [[open-posting-and-friend-visibility]].

## Verification checklist (Apple-facing)

- Block a basic user → their posts leave the home feed. *(works today, incidentally)*
- Block a group co-member → their posts and replies leave the group chat; both of us
  remain participants and can still post.
- Block someone → their profile Posts tab reads empty; their tagged posts leave tag
  pages; a direct link / share code to their post 404s.
- Blocking is symmetric — verify from **both** accounts.
- Report a post → appears in admin filtered by "Reported", resolvable.

## Open questions

1. Should a **reported** post be auto-hidden from the reporter (the X behavior), or
   stay visible until an admin acts? Not specified — currently assumes **stays
   visible**; blocking is the user-side hide lever.
2. Should reports cover **comments/replies** distinctly, or is post-level enough for
   review? The model attaches to `Post`, and replies *are* Posts, so the flag can
   ride on replies for free if the footer row is enabled there ('minimal' variants
   currently render no footer).
3. Do we need a **rate limit** on reports per account per day?
