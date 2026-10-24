---
tags: [nanza-auth, solution-design, auth, social-auth, apple, email]
---

# Apple Private Email Relay — sender registration

> **Status: SETUP CHECKLIST, 2026-08-25.** Companion to
> [[social-identity-providers|Social Identity Providers]]. Needed because the Apple IdP
> mapping saves whatever email Apple provides — including `@privaterelay.appleid.com`
> relay addresses from Hide-My-Email users — and those addresses **bounce ("address not
> found") for any sender Apple doesn't know about.** Verified 2026-08-25 by test.

## How the relay actually works

- A Hide-My-Email user's relay address forwards to their real inbox — but Apple's relay
  servers **only accept mail from domains and addresses registered to your team** in the
  developer portal. Everything else is rejected; the bounce reads "address not found,"
  which looks like a bad address but means *unregistered sender*.
- Registration is checked against **SPF** — the sending domain's DNS has to pass before
  Apple will forward its mail. DKIM should be in order too (it's basic deliverability,
  and Apple weighs it).

## Step by step

### 1. Know what actually sends Nanza email first

Register the *real* sending identities, not guesses:

- [ ] List every from-address users receive mail from (order confirmations, offer
      notifications, Cognito's signup codes if SES-backed, marketing if any).
- [ ] Note the domain each one sends via (e.g. `nanza.app` through SES).
- [ ] Cognito note: if the pool sends its verification emails with **Cognito's default
      email**, that's Apple ↔ AWS infrastructure and can't be registered — relay users
      won't get those until the pool is switched to **SES with a nanza.app from-address**.

### 2. Register in the Apple developer portal

- [ ] developer.apple.com → **Certificates, Identifiers & Profiles**
- [ ] **Services** (left sidebar — NOT Identifiers)
- [ ] **Sign in with Apple for Email Communication** → **Configure**
- [ ] **Email Sources → +** and add:
  - **Domains:** `nanza.app` (covers every from-address on the domain)
  - **Email Addresses:** any senders on *other* domains, individually
- [ ] Apple validates SPF on save — a domain that fails shows as failed here.

### 3. Make sure SPF/DKIM pass

From a terminal:

```bash
dig +short TXT nanza.app          # expect v=spf1 ... include:amazonses.com ... (if SES)
dig +short TXT _dmarc.nanza.app   # DMARC present is a plus
```

- [ ] SPF record includes the actual sender (SES: `include:amazonses.com`)
- [ ] SES domain identity is **verified with DKIM enabled** (SES console → Verified
      identities → nanza.app → DKIM: Successful)
- [ ] If SPF was just fixed, re-check the source in Apple's panel afterward.

### 4. Test correctly

- [ ] Send **from a registered from-address** (e.g. `noreply@nanza.app` via SES) to the
      relay address — this should now land in the user's real inbox.
- [ ] A test from a personal inbox (Gmail, info@oakplatforms.com) **will still bounce**
      — unregistered sender, working as designed. Don't read that as failure.
- [ ] Allow a little propagation time after registering before concluding it's broken.

## Deliverability — what the first live test taught (2026-08-25)

The relay chain verified and forwarded, but the first test landed in **spam**. Fixes
applied, in impact order — all will need repeating at prod if DNS ever diverges:

- **The sending subdomain had no DKIM.** `google._domainkey.nanza.app` existed (parent),
  but DKIM does not inherit — `mail.nanza.app` needed its own key. Generated in Workspace
  Admin (Gmail → Authenticate email → select `mail.nanza.app` → Generate) and published
  at `google._domainkey.mail.nanza.app`. This is the signature that survives Apple's
  relay hop, so it's the main lever against spam placement for relay recipients.
  - **Route 53 quirk:** a 2048-bit key exceeds the 255-char single-string limit — split
    the value into two quoted chunks separated by a space, on one line. Verify from
    outside with `dig +short TXT google._domainkey.mail.nanza.app` — the joined value
    must end `IDAQAB` (a complete RSA key).
- **SPF records had to be created from nothing** — neither `nanza.app` nor
  `mail.nanza.app` published one; every Apple source registration failed until
  `"v=spf1 include:_spf.google.com ~all"` existed at both names. On the bare domain the
  SPF string is a second value INSIDE the existing TXT record set (Route 53 forbids two
  TXT sets at one name); the google-site-verification line stays.
  - If SES ever sends, `include:amazonses.com` joins the SAME record — one SPF record
    per name, providers merged.
- **DMARC exists only at `_dmarc.mail.nanza.app`** (p=none, rua=dmarc@nanza.app), which
  does cover the sending subdomain. **STILL TO DO:** the org-level `_dmarc.nanza.app`
  sibling (same value) — verified absent by dig on 2026-08-25. Also check the
  `dmarc@nanza.app` mailbox actually exists or the aggregate reports silently bounce.
- **Reading a test honestly:** Gmail's ⋮ → Show original prints the SPF/DKIM/DMARC
  verdicts for that exact message — trust those over the folder it landed in. Bare
  one-line tests from a low-volume domain are spam-shaped regardless; reputation builds
  with real transactional volume.
- **Apple's SPF check caches:** a source can stay failed after DNS is fixed — wait a few
  minutes and press Reverify SPF again rather than re-registering.

## Per-environment

One registration serves dev and prod — Apple registers *senders*, not apps or pools. Do
it once; it covers relay users from both environments as long as they're mailed from the
registered domain.

## Related

- [[social-identity-providers|Social Identity Providers]] — the pool + IdP setup this
  extends.
- nanza-mobile [[../../nanza-mobile/solution-designs/auth-screens|Auth Screens]] — why
  relay addresses are in the system at all (the Apple email mapping is deliberately on).
