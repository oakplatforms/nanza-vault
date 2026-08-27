---
tags: [nanza-api, nanza-mobile, moderation, apple, review-response]
---

# App Review Response — Blocking, Reporting & Content Visibility

> **Draft for Apple App Review, 2026-08-23.** Written to accompany the release that adds
> content-level blocking and post reporting. Companion to
> [[blocking-and-reporting|Blocking Visibility & Content Reporting]].

## Response

Nanza is a collectibles community where the feed is **closed by default rather than open**.
There is no public firehose of user posts: a person's posts are visible only to themselves
and to people they have mutually accepted as connections. Connection requests require
consent from both sides, so a user never encounters posts from strangers who happen to be
on the platform. If you are not signed in, you see no member-created content at all — a
signed-out visitor sees only content from affiliates. Once signed in, your feed is exactly
two things: affiliate content, plus posts from the friends you have accepted. This holds
everywhere member content appears — the home feed, group chats, profile pages, tag pages,
and direct links — so there is no surface where unconnected members' content can reach you.

**Affiliates** are the one category of account whose content reaches everyone. These are
businesses and organizations — card shops, publishers, and educational partners — who work
with us directly to produce instructional and market content for collectors. They are
onboarded and vetted by Nanza, the status is assigned by our team and cannot be
self-selected, and they are accountable to us as business partners. Because affiliates are
not personal connections, blocking is not the relevant tool for them; **reporting** is. Any
member can report affiliate content, and those reports go to the same moderation queue as
every other report, where our team reviews them and can remove content or revoke a partner's
affiliate status.

**Blocking and reporting.** Blocking a member is symmetric and removes that person's content
from your experience entirely: their posts and replies disappear from your home feed, from
any group you both belong to, from their profile, and from tag pages and shared links.
Neither of you sees the other, and blocking works this way whether or not you were ever
connected. Inside a group, both people remain members and can keep participating — the
blocked person's content simply no longer renders for the person who blocked them, with no
placeholder revealing the block. Separately, every post carries a report control that opens
a category picker covering spam, hate/abuse/harassment, child safety, violent speech,
graphic or violent media, illegal and regulated behaviors, impersonation, adult sexual
content, private or non-consensual content, and suicide or self-harm, with an optional
free-text field. Reports land in an internal moderation dashboard where our team reviews
each one and can action or dismiss it.

## Notes for the team (not for Apple)

- The "signed out sees only affiliates / signed in sees affiliates + friends" claim is
  literally what `brandVisibilityArms` (nanza-api `validation/post.ts`) implements — the
  guest case drops the friend arm entirely. Verified against the code, not assumed.
- The blocking behavior described here is the release being shipped; before it, blocking
  covered messaging and discovery only. Don't send this until the release is live.
- The reason list matches the shipped `ReportReason` enum one-for-one.
- The moderation dashboard (nanza-admin Posts section) is the last piece still to build —
  confirm it's in the build before this goes out, since the third paragraph promises it.
