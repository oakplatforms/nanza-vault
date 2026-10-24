---
tags:
  - nanza-mobile
  - solution-design
  - analytics
---

# Analytics

How the app reports product events to Mixpanel: one typed catalog, one `track()` gate, and a
`track(EVENTS.X, props)` call at the place that owns each action — a service function right
after its request, or a screen's mount effect.

## The shape: a catalog, one gate, call sites

`src/services/analytics/` is the only module that imports the Mixpanel SDK. Everything else
calls `track(EVENTS.X, props)`.

- **`events.ts`** — the catalog. `EVENTS` is the name map; `EventProps` gives each event its
  prop shape. These are analytics payloads (ids, enums, counts, booleans), never DTOs and
  never PII.
- **`config.ts`** — the levers: `enabled`, the daily cap, the allowlisted spine that always
  sends, per-event caps, the internal-email domains, and the People map (below).
- **`helpers.ts`** — `extractId` (narrow an untyped response to its id), `attachmentTypes` and
  `postPublishedProps` (the post-event props, shared by the two post call sites).
- **`index.ts`** — the gate. `track()`, the session helpers (`identify`, `reset`,
  `applyInternalPolicy`, `setUserProfile`), the `surface` getter, and the pending social
  sign-in.
- **`AnalyticsBridge`** (`components/global/`) — the only component in the picture. It reads
  the session, identifies the user, registers the `surface` getter, and resolves social
  signups.

Every event goes through `track()`: enabled → internal-user policy → rate limiter → `surface`
merge → `mp.track()` → People side effects. Nothing analytics-side can throw into the app:
`track()` is try/catch and returns void.

## Why call sites, not a rules engine (Oct 2026)

Sprint 1 was first built as a declarative rules engine: route rules fed by a navigation
listener, API rules fed from a hook in `fetchData`, and a typed registry of the existing call
sites. It worked, but it cost 696 lines across seven files, an edit to `fetchData`, and a
second mental model next to the eleven `track()` calls the app already had. It was rebuilt as
plain call sites: about 160 lines, no `fetchData` edit, no listener, one pattern. The trade:
each new event means finding its line in product code. The gain: the code reads like the rest
of the app, and "does this fire twice?" is answered by one grep for `track(EVENTS.`. The engine
branch (`mixpanel-rules-engine-s1-10-13-2026`) stays unmerged for reference.

## Where each event fires

Two placements, both in use before S1:

- **Service functions**: `const res = await fetchData(...)`, `track(...)`, `return res`. The
  request has succeeded (`fetchData` throws otherwise) and the response is in hand. This is
  where writes are tracked.
- **Screens**: `useEffect(() => { track(...) }, [])` fires once per mount. React Navigation keeps
  a pushed screen mounted, so going back does not re-fire a view event, and a fresh push does.
  This is where views are tracked.

| Event | Fires in | Note |
| --- | --- | --- |
| `post_published` | `postService.create` (non-draft), `postService.publish` | `create` covers the composer publish, both reply paths and the audience-changed draft (`from_draft: false`). `publish` carries `post_id` and `from_draft: true` only — the endpoint receives just `{ accountId }` |
| `post_draft_saved` | `postService.create` (`draft: true`) | a re-dismiss of the same draft is a PUT (`updateFull`) and fires nothing |
| `post_liked` | `postService.toggleLike` | only when the response says `liked`; unlike is silent |
| `post_reported` | `reportService.createForPost` | a repeat report 409s and throws, so nothing fires |
| `post_viewed` | `PostDetailScreen` mount | `author_is_affiliate` from `post.account.profile.type`, `is_own` against the session account, `is_reply` from `parentId` |
| `tag_page_viewed` | `SupportedTagValueScreen` mount | named for the object (a supported tag value: class, hero, rarity…), not "brand page" — brands are their own type. A child value is a fresh push |
| `profile_viewed` | `UserProfileScreen` mount | skipped when the `profileId` is the viewer's own (covers the Profile tab) |
| `search_result_clicked` | `SearchEntityRow` (push branch), the group row `onPress` in `SearchScreen` | `result_type` is the route opened. Entity taps inside `CollectionSearchOverlay` reach the same push branch; `surface` tells them apart |
| `signup_completed` (social) | `AnalyticsBridge` | see "Social signups" |

The pre-existing call sites are unchanged: `install` / `app_open` (`lifecycle.ts`), the email
`signup_completed` (`ConfirmationCode`), `search_executed` (`SearchScreen`), `product_view`
(`EntityScreen` mount), the four `offer_*` events (`MakeOfferScreen`,
`OfferMessageDetailScreen`), `listing_published` (`CreateListingScreen`), `group_joined`
(`Group.ts`), `message_sent` (`Message.ts`), `scan_completed` (`Scan.ts`), `payout_completed`
(`Seller.ts`).

**`post_published` props come from the create payload, not the response.** At the service
boundary the payload is still the JS object even when the request goes multipart, so
`contentItems`, `supportedTagValueIds`, `parentId` and the audience are all readable there.
`attachment_types` is the content item types minus `RICH_TEXT`, with GROUP galleries flattened
to their children; `has_text` is a `body` or a `RICH_TEXT` item. Body text never goes into an
event. What the composer's flows produce:

- Dismiss with content, no draft yet → `post_draft_saved`.
- Dismiss again, same home → `PUT /post/:id` → nothing.
- Publish a restored draft, same home → `PUT …/publish` → one `post_published`, `from_draft: true`.
- Publish after changing audience → `POST /post` + `DELETE` → one `post_published`, `from_draft: false`.
- Reply (composer or thread) → `post_published` with `is_reply: true`.

## Automatic `surface`

`track()` merges `surface: <focused route name>` into every event. `AnalyticsBridge` registers a
getter (`setSurfaceGetter`) that reads `navigationRef.getCurrentRoute()?.name` **at track
time**, guarded by `isReady()`. There is no listener and no cached route: React Navigation's
container writes its state store synchronously on dispatch and emits the `state` event later,
from an effect, so a mount effect in the screen that was just pushed already sees itself as the
current route. A listener-fed cache would lag one screen behind for every mount-effect event
(`product_view` included) because child effects run before the container's. Before the
container is ready the getter returns `undefined`, and the key is dropped on serialization
(`mixpanel-react-native` 3.3.0 is pure JS). `surface` is a `string`, not a `RootStackParamList`
key: the focused route is often a tab inside `Main` (`Home`, …).

## People properties

`config.people` maps an event to profile side effects, applied in `track()` after the event has
passed every gate:

| Event | setOnce (ISO) | increment | contribution |
| --- | --- | --- | --- |
| post_published | first_post_at | posts_count | yes |
| listing_published | first_listing_at | listings_count | yes |
| offer_sent | first_offer_sent_at | offers_sent_count | yes |
| message_sent | | messages_count | yes |
| group_joined | first_group_joined_at | groups_joined_count | |
| scan_completed | first_scan_at | scans_count | |

`contribution` also sets `last_contribution_at`. There is no "identified" guard: every mapped
event needs a signed-in session, and `identify()` ran when that session loaded.
`mixpanel-react-native` 3.3.0 People signatures: `set(prop, to)` / `set(object)`,
`setOnce(prop, to)`, `increment(prop, by)`.

## Social signups

`signInWithApple` / `signInWithGoogle` resolve `isSignedIn: true` for new and returning users
alike, so the button cannot tell a signup from a login. `SocialAuthButtons` only marks the
method (`markPendingSocialSignIn`). `AnalyticsBridge` consumes the mark once the account row has
loaded and `identify()` has run, and fires `signup_completed { method }` when
`account.createdAt` is under ten minutes old. The email path keeps its own call in
`ConfirmationCode`.

## Catalog and limits (S1 state)

- Removed, no call sites: `cart_add`, `checkout_started`, `order_placed`, `trade_proposed`,
  `trade_accepted`.
- Allowlist (always sends): `install`, `signup_completed`, `payout_completed`,
  `post_published`, `offer_sent`.
- Per-event daily caps: `search_executed` 200, `product_view` 300, `message_sent` 200,
  `post_viewed` 300, `post_liked` 200, `tag_page_viewed` 200, `profile_viewed` 200,
  `search_result_clicked` 200. Daily total 500.
- Internal users (`oakplatforms.com`) are tagged `is_internal`, kept in the Dev project and
  dropped in Prod (`MIXPANEL_ENV === 'prod'`). The verdict is set at `identify()`, so the
  cold-start `install` / `app_open` from a team device still reach Prod; persisting the verdict
  per device was prototyped on the engine branch and is a separate, small change if wanted.

## Lookup tables: ids on events, names at query time

The operating guide — how to map the table, which boards to build and how to read each event —
is the shared [[../../_shared/architecture/mixpanel-playbook|Mixpanel playbook]].

Events carry ids only (no usernames, no post text), and a Mixpanel **Lookup Table** resolves
them when a report is read. One table, **"Nanza objects"**, keyed by `id`, with columns
`kind` (Card / Post / Profile / Account / Group / Tag / Brand), `label`, `game`, `detail`,
`created`. It is mapped in Lexicon to every id property — `post_id`, `entity_id`,
`supported_tag_value_id`, `profile_id`, `group_id`, and the polymorphic `target_id` /
`result_id` — so any id column in any report shows "by Bios_Update: I'm beyond HYPED…" or
"Anticipating Gaze · Hala · AHA003-RF" next to it. A non-technical reader never sees a bare id.

**Why a lookup table and not properties on the event.** Mixpanel bills per event, so extra
properties cost nothing; the reasons are governance. A username on another user's event can't
be purged when that user deletes their account (Mixpanel deletes by distinct id); a username on
an event goes stale on rename; post text is user content and long strings are truncated on
events anyway. A lookup row is current, deletable, and lives in one place. The post `label`
does include the first 80 characters of the post's text — a deliberate choice (Oct 2026): only
people with project access see it, and the row can be removed.

**Where the rows come from.** No database access is needed: the public API, read with the same
guest token the app uses, returns every object by id (`/post/:id`, `/profile/:id`, `/entity/:id`
with brand/set/product, `/supported-tag-value/:id`, `/group/:id`, `/brands`, `/groups`). Rows
are built for the ids Mixpanel has actually seen (one `Get-Property-Values` per id property),
plus the full brand and group catalogs, so private-group posts and any profile resolve without
crawling feeds, and the table stays small (hundreds of rows, not the 16k-card catalog). The
Dev project receives events from builds pointed at either API, so Dev rows are built from
**both** `api.nanza.app` and `api-dev.nanza.app` (ids are globally unique cuids, so the union
is safe); Prod rows come from the production API only.

**Known gaps.** A guest token cannot read posts in private groups or deleted posts, so those
ids stay unresolved (they show as the raw id). Resolving private-group posts would need an
authenticated or admin read, which is the one case where an API-side export earns its keep.

**Refresh.** Manual on request to begin with (the build script runs the property-value reads
and API fetches, then upserts the delta through the Mixpanel MCP). Once the shape is validated
with the reader, a nightly scheduled routine repeats the same steps.

## Known limits (accepted)

- A cold-start `app_open` fires before the navigator is ready and carries no `surface`.
- A draft dismissed to a different home posts again and fires a second `post_draft_saved`.
- The email signup still fires before `identify()`, as before.
- Cold-starting straight into the Profile tab before the session has loaded fires one
  `profile_viewed` for the viewer's own profile.
- Entity taps in the collection search overlay count as `search_result_clicked`
  (`surface` distinguishes them from the Search screen).

## Verification

TypeScript and ESLint clean (the three new mount effects carry the same `exhaustive-deps`
warning as the existing `product_view` effect). The on-device pass in Mixpanel Dev
(project 4029527, Live View) is still to be run:

1. Cold start on Home, open a post → `post_viewed` with `surface: PostDetail`.
2. Fresh Apple / Google account → one `signup_completed` with the method; a repeat sign-in → none.
3. Root post with a photo and a listing → one `post_published`, `from_draft: false`,
   `attachment_types: [IMAGE, LISTING]`, `has_text`, `tag_count`, `surface: Home`; profile
   `first_post_at`, `posts_count`, `last_contribution_at`.
4. Draft: dismiss with content → one `post_draft_saved`; re-dismiss unchanged → nothing;
   publish → one `post_published`, `from_draft: true` (`post_id` and `from_draft` only).
5. Restored draft, audience changed to a group → `post_published`, `from_draft: false`,
   `target_type: GROUP`.
6. Reply → `post_published`, `is_reply: true`, `attachment_count: 0`, `surface: PostDetail`.
7. Like, unlike, like → `post_liked` on the first and third taps only.
8. Report → `post_reported` with the reason, `surface: ReportPost`; a repeat report → nothing.
9. Affiliate post, regular post, own post → `post_viewed` each with the right flags; back from
   the author's profile → no second event; reopening from the feed → a new one.
10. Tag page, then a child value → two `tag_page_viewed`.
11. Another user's profile → `profile_viewed`; own profile (post avatar or Profile tab) → nothing.
12. Search → card result → `search_executed` (`surface: Search`), `search_result_clicked`
    (`Entity`), one `product_view`; group result → `GroupDetailScreen`; Entity → Search → back
    → nothing new.
13. Existing call-site events once each, with `surface` (`product_view` now carries `Entity`,
    not the screen before it); profile counters move for message, listing, scan, group join.
14. Two more posts → `posts_count` +2, `first_post_at` unchanged, `last_contribution_at` updated.
15. `enabled: false` → identical behaviour, nothing in Live View.
