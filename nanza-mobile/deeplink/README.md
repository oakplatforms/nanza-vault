# Deep-link association files — hosting handoff

These two files make `https://nanza.app/<referenceCode>` open the native app
(iOS Universal Links + Android App Links), falling back to the web page when the
app isn't installed. The app-side wiring (entitlement, intent-filter, JS routing)
is already in the mobile repo — these two files are the **infra** half and must be
hosted on `nanza.app`.

See the full design: `../REFERENCE.md` and the partner doc the native steps came from.

## What to host

| File in this folder | Serve at | Content-Type |
| --- | --- | --- |
| `apple-app-site-association` | `https://nanza.app/.well-known/apple-app-site-association` | `application/json` |
| `assetlinks.json` | `https://nanza.app/.well-known/assetlinks.json` | `application/json` |

Hard requirements (links silently fail to open the app otherwise):
- HTTPS with a valid, non-self-signed certificate.
- **No redirects** on either path (a 301/302 breaks Apple's fetch).
- The AASA file has **no file extension** — serve the file named exactly
  `apple-app-site-association`, not `.json`.

## Certificate fingerprints (both filled in)

`assetlinks.json` lists **both** fingerprints, which is intentional:

1. **Play App Signing key** (`41:53:...:66:67`) — Play re-signs store builds with this
   key, so it's the one that matters for production/internal-testing installs.
2. **Upload key** (`E8:AF:...:DB:29`) — covers release builds installed directly
   (sideloaded) for local testing.

The Play App Signing SHA-256 was extracted from the Play-signed universal APK
(App bundle explorer → Downloads → "Signed, universal APK"), whose certificate DN
is `O=Google Inc.`. It also appears in **Play Console → App integrity → App signing
key certificate** if that page is accessible for your role.

(The Apple side needs no secret: Team ID `Y9X5CD9HAV` + bundle `com.oakplatforms.nanza`
are already filled in.)

## Scope: 6-char codes only

Matching is intentionally narrowed to a single 6-character path segment
(`/??????` on iOS, `pathPattern="/......"` on Android) so marketing routes stay in
the browser. Reference codes are 6 chars (5 digits 2–9 + one type letter S/B/C/P/K).
The app's `getStateFromPath` re-validates the exact code shape and falls through for
anything else.

## Verify after hosting

```bash
# Both return 200, JSON, no redirect:
curl -i https://nanza.app/.well-known/apple-app-site-association
curl -i https://nanza.app/.well-known/assetlinks.json

# Google's Android verifier (expects a matching statement):
curl "https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://nanza.app&relation=delegate_permission/common.handle_all_urls"
```

On-device (after installing a fresh build that includes the native changes):
- Android: `adb shell am start -a android.intent.action.VIEW -d "https://nanza.app/32S392"`
  then `adb shell pm get-app-links com.oakplatforms.nanza` → should show `verified`.
- iOS (real device, not simulator): paste `https://nanza.app/32S392` into Notes,
  long-press → **Open in "nanza"**.

> Association files are only fetched on install, so they must be live **before** the
> deep-link build is distributed.
