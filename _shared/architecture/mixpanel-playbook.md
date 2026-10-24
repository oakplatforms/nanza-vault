---
tags:
  - shared
  - architecture
  - analytics
  - mixpanel
---

# Mixpanel playbook: from ids to insight

How to set up and read the Nanza Mixpanel projects so that every event is measurable and a
non-technical reader can open a board and understand it without asking an engineer. Written
for the Oak team operating Mixpanel; the app-side design (what fires when, and why) is in
[[../../nanza-mobile/solution-designs/analytics|nanza-mobile › Analytics]].

Projects: **Dev** `4029527` (internal traffic kept, tagged `is_internal`) and **Prod**
`4029528` (internal traffic dropped). Everything below is set up in Dev first, verified, then
repeated in Prod.

## 1. The model in one minute

- **Events carry ids, never names or text.** `post_id`, `entity_id` (a card), `profile_id`,
  `group_id`, `supported_tag_value_id` (a tag page), `author_account_id`, plus two
  polymorphic ones: `target_id` (where a post was published, typed by `target_type`) and
  `result_id` (what a search tap opened, typed by `result_type`).
- **One lookup table, "Nanza objects", turns ids into names at read time.** Key `id`;
  columns `kind` (Card / Post / Profile / Account / Group / Tag / Brand), `label`, `game`,
  `detail`, `created`. Once a property is mapped to it, every report can break down by
  `label` instead of the id.
- **Every event carries `surface`** (the screen the user was on), `platform`, `app_version`
  and `is_internal`.
- **People profiles carry the user's firsts and counts:** `first_post_at` / `posts_count`,
  `first_listing_at` / `listings_count`, `first_offer_sent_at` / `offers_sent_count`,
  `messages_count`, `first_group_joined_at` / `groups_joined_count`, `first_scan_at` /
  `scans_count`, and `last_contribution_at`. These power cohorts without any event math.
- **Identity:** `distinct_id` is the account id. Anonymous `install` / `app_open` stitch to the
  account at signup.

## 2. One-time setup in the UI (in this order)

### 2.1 Map the lookup table

Mixpanel offers two kinds of mapping. Both use the same table.

- **Persistent (Lexicon).** Growth plan or above, Admin/Owner role; the Startup program runs on
  Growth features, so it should be available. The prompt only appears inside Mixpanel's own
  import flow, so: Lexicon → Lookup Tables → **Nanza objects** → **Export Lookup Table**
  (download the CSV) → **Replace Lookup Table** → upload that same CSV → at the step "Map the
  Lookup Table to an event or user profile property" pick `post_id`. Repeat for `entity_id`,
  `profile_id`, `group_id`, `target_id`, `result_id`, `supported_tag_value_id`,
  `author_account_id` (the last two only appear after their events have fired at least once).
  The table page then shows **Properties Linked: 8**.
- **Per report.** Any plan. In a report, add a breakdown by the id property, click the **"…"**
  next to it → **Map Lookup Table** → choose **Nanza objects**. The mapping is saved with that
  report, so boards built this way work for every viewer; it just has to be repeated per
  report.

If the persistent prompt never appears, the plan doesn't include it: build the reader's boards
with per-report mappings instead (section 4) and nothing is lost.

### 2.2 Lexicon hygiene

- **Hide retired names** so the reader never meets them: `brand_page_viewed` (test name for
  `tag_page_viewed`), and any `cart_add`, `checkout_started`, `order_placed`, `trade_proposed`,
  `trade_accepted` still listed (they never fired from the app).
- **Describe every event** in one sentence stating the moment it fires. The S1 events are
  undescribed in Lexicon today; the sentences are in section 3.
- **Tag events by loop:** `Acquisition` (install, signup_completed), `Discovery`
  (search_executed, search_result_clicked, product_view, tag_page_viewed, profile_viewed,
  post_viewed), `Publishing` (post_published, post_draft_saved, post_liked, post_reported),
  `Community` (group_joined, message_sent), `Marketplace` (listing_published, offer_*,
  payout_completed, scan_completed), `Retention` (app_open). Tags become the reader's filter
  when browsing events.
- **Mark `$email`, `$name`, `$phone` on profiles as sensitive** (Data Governance), so they
  never show up in breakdowns.
- **Turn on Data Volume Monitoring** (the banner on Event Properties) and set it on
  `signup_completed`, `post_published`, `listing_published` and `app_open`: a silent drop
  after a release is the failure mode to catch.

### 2.3 Custom properties for readable dimensions

A handful of formulas turn enum codes into words the reader can filter by. Create these once
in Lexicon → Custom Event Properties:

| Custom property | Formula (illustrative) | Used on |
| --- | --- | --- |
| Post kind | `if(is_reply == true, "Reply", "Post")` | post_published, post_viewed |
| Publish path | `if(from_draft == true, "From a draft", "Direct")` | post_published |
| Posted to (kind) | `if(target_type == "BRAND", "Brand feed", if(target_type == "GROUP", "Group", if(target_type == "SUPPORTED_TAG_VALUE", "Tag page", target_type)))` | post_published |
| Search result kind | `if(result_type == "Entity", "Card", if(result_type == "GroupDetailScreen", "Group", result_type))` | search_result_clicked |
| Has attachments | `attachment_count > 0` | post_published |
| Author kind | `if(author_is_affiliate == true, "Affiliate", "Member")` | post_viewed |

### 2.4 Custom events for funnels

- **first_contribution** = `post_published` OR `listing_published` OR `offer_sent` OR
  `message_sent`. The activation moment; used in the funnel in section 4.
- **discovery_open** = `product_view` OR `post_viewed` OR `tag_page_viewed` OR
  `profile_viewed`. "Opened something" after a search or a feed.

### 2.5 Cohorts

| Cohort | Definition | Why |
| --- | --- | --- |
| Internal | profile or event `is_internal = true` | Exclude from everything in Dev (Prod already drops them). |
| New this week | did `signup_completed` in the last 7 days | Activation reads. |
| Contributors | `posts_count > 0` or `listings_count > 0` or `offers_sent_count > 0` or `messages_count > 0` | The people who make the marketplace. |
| Lapsed contributors | `last_contribution_at` older than 14 days and Contributors | Re-engagement target. |
| Affiliates | `account_type` / profile type affiliate | Their posts drive discovery. |
| Lurkers | `app_open` ≥ 3 in 7 days and not Contributors | The conversion opportunity. |

### 2.6 Business context (for Mixpanel's own AI features)

Mixpanel Agent and the query tools read the project's business context. It still describes the
pre-S1 catalog. Replace it with the text in the appendix (ask Claude to apply it through the
Mixpanel connection, or paste it under Settings → Project → Business context).

## 3. Every event, the question it answers, and how to read it

| Event | Fires when | Question | Report | Breakdown / filter |
| --- | --- | --- | --- | --- |
| install | first launch after install | How many installs? | Insights, unique users, weekly | platform |
| app_open | cold start or genuine resume | Are people coming back? | Retention: app_open → app_open, weekly | cohort New this week |
| signup_completed | account created (`method` email / apple / google) | Where do signups come from? | Insights, total, weekly | method, platform |
| search_executed | a query ran | What do people look for? Do they find it? | Insights total; filter `result_count = 0` for dead ends | query (top 20), surface |
| search_result_clicked | a result tapped from Search | What wins the click? | Insights total | Search result kind, Search result → label |
| product_view | card detail opened | Which cards get attention? | Insights total, top 20 | Card → label, Card → game |
| tag_page_viewed | a class / hero / rarity page opened | Which tag pages matter? | Insights total | Tag page → label |
| profile_viewed | someone else's profile opened | Who gets looked at? | Insights total | Profile viewed → label |
| post_viewed | a post opened (first focus only) | Which posts and authors draw views? | Insights total | Post → label, Post author → label, Author kind |
| post_published | a post went live | How much is being posted, where, how? | Insights total, weekly | Posted to (kind), Publish path, Has attachments, attachment_types |
| post_draft_saved | composer dismissed with content | How often do drafts start? | Insights total vs post_published | surface |
| post_liked | a like (unlike is silent) | What resonates? | Insights total | Post → label |
| post_reported | a report submitted | Moderation load and reasons | Insights total | reason, Post → label |
| group_joined | membership became ACTIVE | Are groups growing? | Insights total | Group → label |
| message_sent | a DM sent | Is the community talking? | Insights unique users, weekly | surface |
| listing_published | listing went live | Is supply growing, and how is it created? | Insights total, weekly | creation_method, source |
| offer_sent / accepted / declined / canceled | the offer lifecycle | Do offers convert? | Funnel offer_sent → offer_accepted (7 days) | offer_type |
| payout_completed | seller paid | Is money moving? | Insights sum of amount, weekly | – |
| scan_completed | a scan finished | Is scan the on-ramp it should be? | Insights total; funnel scan_completed → listing_published | game, matched_count |

Reading rules that trip people up:

- In **Dev**, always add the filter `is_internal = false` (or the cohort). Prod has no
  internal traffic.
- **Rate caps** exist (500 events per user per day, lower per-event caps on the high-volume
  ones). Very heavy testers undercount; real users don't hit them.
- **`product_view`'s `surface` is the screen the user came from**, not Entity: it fires on
  mount before the navigator reports the new route. Every rule-fired event has the right
  surface.
- **`post_viewed` fires once per visit to a post.** Going back to the same post (still in the
  stack) does not fire again; opening it again from the feed does.
- **`post_liked` counts likes only.** An unlike is silent, so a like/unlike/like is two events.
- **Draft flows:** dismissing the composer with content fires `post_draft_saved`; publishing
  that draft later fires one `post_published` with `from_draft = true`. Changing the audience
  before publishing makes it a direct publish (`from_draft = false`).
- **People counts increment per event**, so `posts_count` is lifetime posts, not posts in the
  period. Use events for period counts and People properties for cohorts.

## 4. The boards the reader opens

Build these in Dev with per-report lookup mappings, then recreate in Prod. Five boards, each
answerable in one glance; the first is the only one that needs to be looked at daily.

### Board 1 — Nanza pulse (daily)
1. **Signups this week** — `signup_completed`, total, this week vs last, breakdown `method`.
2. **Active people** — `app_open`, unique users, daily, 30 days.
3. **Contributions** — custom event `first_contribution`, total, weekly, 12 weeks.
4. **Activation funnel** — `install` → `signup_completed` → `first_contribution`, 14-day
   window, conversion over time.
5. **Retention** — `app_open` weekly retention, cohort New this week.

### Board 2 — Publishing
1. `post_published` weekly, broken down by **Posted to (kind)**.
2. `post_published` by **Publish path** (drafts are a real behaviour or not).
3. `post_published` by `attachment_types` (which attachments people bother with).
4. `post_viewed` top 20 by **Post → label** (what is being read).
5. `post_viewed` by **Author kind** (are affiliates carrying the feed).
6. `post_liked` top 10 by **Post → label**; `post_reported` by `reason`.

### Board 3 — Discovery
1. **Search funnel** — `search_executed` → `search_result_clicked` → `product_view`, same
   session.
2. **Dead-end searches** — `search_executed` with `result_count = 0`, broken down by `query`.
3. **Top cards** — `product_view` top 20 by **Card → label**, with **Card → game** as a filter.
4. **Tag pages** — `tag_page_viewed` by **Tag page → label**.
5. **Profiles** — `profile_viewed` by **Profile viewed → label**.

### Board 4 — Community
1. `group_joined` by **Group → label**, weekly.
2. `message_sent` unique senders, weekly.
3. Cohort sizes: Contributors, Lurkers, Lapsed contributors (Insights on cohort membership).

### Board 5 — Marketplace
1. `listing_published` weekly by `creation_method`.
2. **Scan on-ramp** — `scan_completed` → `listing_published`, 1-day window.
3. **Offer conversion** — `offer_sent` → `offer_accepted`, 7-day window, by `offer_type`.
4. `payout_completed` sum of `amount`, monthly.

Naming: boards are `Nanza · <Board>`; reports inside are a short question ("Which cards get
attention?"), not an event name. The reader should never have to translate.

## 5. Keeping the lookup table fresh

Rows are built for the ids Mixpanel has actually seen (one property-values read per id
property), fetched from the public API by id with the same guest token the app uses, plus the
full brand and group catalogs. No database access, no new service. Dev rows come from both
`api.nanza.app` and `api-dev.nanza.app` (Dev receives events from builds pointed at either);
Prod rows from the production API only.

- **Today:** ask Claude to refresh. It reads new ids from Mixpanel, fetches them, and upserts
  the delta through the Mixpanel connection (adds rows; existing rows keep their values).
- **Manual alternative:** export the table from Lexicon, add rows to the CSV, **Replace Lookup
  Table**. The replace is wholesale, so start from the export.
- **Next:** a nightly scheduled routine that repeats the same steps.

What stays unresolved: posts in private groups and deleted posts (a guest can't read them),
and profiles of accounts that never posted publicly. They show as raw ids. Resolving private
group posts would need an authenticated read, which is the one place an API-side export would
earn its keep.

## 6. Dev versus Prod

| | Dev 4029527 | Prod 4029528 |
| --- | --- | --- |
| Internal users | kept, `is_internal = true` | dropped; a team device stays silent from its first event once anyone internal has signed in on it |
| Lookup rows | from both APIs | from the production API |
| Boards | build and verify here | recreate; boards don't copy across projects |
| Business context | updated for S1 | update once the S1 build ships |

A test build's `.env` decides which Mixpanel project and which API it talks to; keep the two
aligned (Dev Mixpanel with either API is fine; Prod Mixpanel only with the production API and a
release build).

## 7. Gaps worth closing (need code, not UI)

1. **No purchase events.** `cart_add`, `checkout_started` and `order_placed` never fired and
   were removed. The buyer conversion moment is unmeasured. The rules engine makes this a
   two-line API rule once the order endpoint and its response are confirmed.
2. **No game dimension.** The selected brand is session state, not a super property. A `game`
   super property would let every board filter Flesh and Blood vs One Piece without a lookup.
3. **Migrate call-site events to rules.** `product_view` (route), the offer lifecycle,
   `listing_published`, `scan_completed`, `group_joined`, `message_sent`, `payout_completed`
   (API). Each move is: delete the `track()` call, drop the registry entry, add the rule.
4. **Seller onboarding** (`seller_onboarding_submitted`) exists in the old business context
   but not in the app catalog; confirm whether the web app sends it, or add it.

## 8. Weekly ritual

Monday, ten minutes, Board 1: signups (and method mix), active people, contributions,
activation conversion, retention of last week's cohort. If a number moved, open the matching
board (2 to 5) for the why. If an event is flat at zero after a release, check Data Volume
Monitoring and the app's `[analytics]` log before assuming behaviour changed.

## Appendix — business context text (paste or apply)

```
# Nanza — Business Context (DEV project 4029527)
Nanza is a React Native marketplace for TCG cards (Flesh and Blood, One Piece), built by Oak Platforms. Internal/test traffic IS present here (is_internal = true); exclude it when analysing real behaviour.

## Identity and super properties
distinct_id = account.id, identified on login, reset on logout. Every event carries platform, app_version, is_internal, and surface (the screen the user was on).

## Reading ids
Events carry ids, never names or text. The lookup table "Nanza objects" resolves post_id, entity_id (card), supported_tag_value_id (tag page), profile_id, group_id, author_account_id, target_id (where a post was published) and result_id (what a search tap opened) to kind / label / game / detail / created. Use the lookup columns, not the raw id.

## Events
Lifecycle: install, app_open, signup_completed (method: email | apple | google).
Discovery: search_executed, search_result_clicked (result_type, result_id), product_view (entity_id), tag_page_viewed, profile_viewed (never the viewer's own), post_viewed (author_account_id, author_is_affiliate, is_own, is_reply, post_created_at).
Publishing: post_published (from_draft, is_reply, target_type, target_id, attachment_types, attachment_count, tag_count, has_text), post_draft_saved, post_liked (like only; unlike is silent), post_reported (reason).
Marketplace: offer_sent / offer_accepted / offer_declined / offer_canceled, listing_published (creation_method scan|manual), payout_completed, group_joined, message_sent, scan_completed.
Removed Oct 2026 (never fired): cart_add, checkout_started, order_placed, trade_proposed, trade_accepted.

## People properties
first_post_at / posts_count, first_listing_at / listings_count, first_offer_sent_at / offers_sent_count, messages_count, first_group_joined_at / groups_joined_count, first_scan_at / scans_count, last_contribution_at.

## Gotchas
Rate caps: 500/user/day, per-event caps on high-volume events; install, signup_completed, payout_completed, post_published, offer_sent bypass caps. product_view's surface is the screen the user came from (fires on mount before the route updates). brand_page_viewed is a retired test name for tag_page_viewed.
```
