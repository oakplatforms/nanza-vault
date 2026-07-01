# APNs Auth Key — iOS Push Delivery (your side)

This is the one Apple-side piece push needs. The **APNs Authentication Key** (a `.p8` file) lets
Firebase deliver pushes to iPhones. Without it: Android works, **iOS silently delivers nothing**.

- You generate it **once** in your Apple Developer account.
- You upload it **once** to Firebase.
- It covers **both dev and prod** — a single APNs key works for all environments. No per-stage key.

Requires: Apple Developer Program membership (you already have this to ship iOS).

Values you'll need (have them handy):
| Thing | Where |
|---|---|
| App bundle ID | `com.oakplatforms.nanza` |
| Firebase project | `nanza-a1ff7` |
| Apple Team ID | Apple Developer → top-right, or Membership page (10 chars, e.g. `AB12CD34EF`) |

---

## Step 1 — Create the APNs key in Apple (~3 min)
1. Go to https://developer.apple.com/account → **Certificates, Identifiers & Profiles**.
2. Left menu → **Keys** → click the **＋** (Create a key).
3. **Key Name:** `nanza APNs` (any name).
4. Check the box **Apple Push Notifications service (APNs)**.
5. **Continue → Register.**
6. **Download** the key — a file named `AuthKey_XXXXXXXXXX.p8`.
   - ⚠️ You can only download it **once**. Keep it safe (don't commit it to git).
   - The `XXXXXXXXXX` part is the **Key ID** — note it (also shown on the key's page).

## Step 2 — Confirm the App ID has Push enabled (~2 min)
1. Same site → **Identifiers** → click your app `com.oakplatforms.nanza`.
2. Make sure **Push Notifications** is checked/enabled. If not, enable it and Save.
   (This is usually already on once the app exists, but confirm.)

## Step 3 — Get your Team ID (~1 min)
- Top-right of the Apple Developer site, or **Membership** page. It's a 10-character string.

## Step 4 — Upload the key to Firebase (~3 min)
1. Go to https://console.firebase.google.com → project **nanza-a1ff7**.
2. ⚙️ **Project settings → Cloud Messaging** tab.
3. Find **Apple app configuration** → your iOS app (`com.oakplatforms.nanza`).
4. Under **APNs Authentication Key** → click **Upload**.
5. Provide:
   - The `.p8` file from Step 1
   - **Key ID** (the `XXXXXXXXXX` from the filename / key page)
   - **Team ID** (from Step 3)
6. **Upload / Save.**

Done. Firebase can now deliver to iOS for both dev and prod builds.

---

## How to verify it's working (after the app + API are deployed)
- Run the app on a **real iPhone** (push does NOT work on the iOS Simulator — it has no APNs).
- Sign in (registers the device token), then have another account send you a message/offer.
- The push should arrive. If Android works but iOS doesn't, the APNs key is the usual culprit —
  re-check the Key ID / Team ID / that the `.p8` was uploaded to the right project.

## Notes
- **One key, all environments.** The same APNs key serves dev and prod. Don't make a second one.
- **Simulator caveat.** iOS push only works on physical devices. Test iOS on a real phone.
- **Keep the `.p8` private.** It's a credential — store it somewhere safe, never in the repo.
