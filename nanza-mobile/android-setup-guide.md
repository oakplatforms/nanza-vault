# Android Setup Guide — Nanza Mobile

Everything you need to get the Nanza Mobile Android build running locally on your machine. Covers environment setup, Android Studio configuration, emulator creation, and a snapshot of where the Android side of the project currently stands.

---

## Part 1 — Local Environment Setup

Add these three lines to the bottom of `~/.zshrc`, then run `source ~/.zshrc` (or open a new terminal).

```bash
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator:$ANDROID_HOME/platform-tools:$ANDROID_HOME/cmdline-tools/latest/bin
export JAVA_HOME=/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
```

### Verify

```bash
echo $ANDROID_HOME      # should print /Users/<you>/Library/Android/sdk
echo $JAVA_HOME         # should print /Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
java -version           # should print "openjdk version "17..."
adb --version           # works only after SDK Manager finishes installing platform-tools
```

### Prerequisites

You'll need:

1. **JDK 17** (RN 0.73+ requires 17). Recommended: `brew install --cask zulu@17` or Android Studio's bundled JDK.
2. **Android Studio** (for SDK Manager, AVD Manager, and the bundled SDK + platform-tools).

---

## Part 2 — Android Studio Setup

### Step 1 — SDK Manager: install platform + tools

Open SDK Manager (gear icon on welcome screen → SDK Manager).

**SDK Platforms** tab:
- ✅ Check **Android 14.0 ("UpsideDownCake")** = API 34

**SDK Tools** tab:
- ✅ Check **Show Package Details** (checkbox in bottom right) — required to pick specific versions
- Expand **Android SDK Build-Tools 34** → check **34.0.0**
- Expand **NDK (Side by side)** → check **25.1.8937393**
- Expand **CMake** → check the latest version (e.g. 3.22.x)
- Also install **Android SDK Command-line Tools** (latest)

Click **Apply** → accept licenses → wait for download to finish.

### Step 2 — AVD Manager: create an emulator

Welcome screen → gear icon → **Virtual Device Manager** (or, if a project is open: **Tools → Device Manager**).

1. Click **+ Create Virtual Device**
2. Pick **Pixel 7** → **Next**
3. **System Image** screen → pick **UpsideDownCake (API 34)** → if there's a download arrow next to it, click it first → **Next**
4. **Finish**
5. Back in the device list, click the **▶** play button next to your new Pixel 7 to boot it

The emulator boots in a separate window — wait until you see the home screen.

### Step 3 — Bump Android Studio + emulator memory (recommended on Apple Silicon)

Out of the box Android Studio ships with conservative memory settings — on Apple Silicon Macs (M1/M2/M3/M4) the IDE, Gradle daemon, and emulator will all feel sluggish until you raise them. Do this once before your first build.

**1. Android Studio IDE heap**

Android Studio → **Settings** (or **Preferences**) → **Appearance & Behavior** → **System Settings** → **Memory Settings**.

- **IDE max heap size:** raise from the default (usually 2048 MB) to **4096 MB** (or 8192 MB if you have 32 GB+ of RAM).
- Click **Save and Restart**.

**2. Gradle daemon heap**

Already wired up in this project — `android/gradle.properties` has `org.gradle.jvmargs=-Xmx4096m`. No action needed unless you see `OutOfMemoryError` during a Gradle build, in which case bump it to `-Xmx6144m` or `-Xmx8192m`.

**3. Emulator RAM + CPU cores**

Android Studio → **Device Manager** → click the **pencil/edit icon** next to your Pixel 7 AVD → **Show Advanced Settings** (button at the bottom).

- **Memory and Storage → RAM:** raise from the default (often 1536 MB or 2048 MB) to **4096 MB**.
- **Memory and Storage → VM heap:** raise to **512 MB**.
- **Multi-Core CPU:** set to **4 cores** (or however many physical cores your Mac can spare — on an M-series chip 4 is a safe default).
- **Graphics:** set to **Hardware - GLES 2.0** (uses your Mac's GPU).

Click **Finish**, then cold-boot the emulator (Device Manager → ▼ menu next to the device → **Cold Boot Now**) so the new settings take effect.

**Why this matters:** the default settings are tuned for low-end laptops. On a modern Mac the emulator is the main bottleneck for `npm run android` cycle time — a properly-sized AVD boots in under 30 seconds and runs the app at near-native speed. An undersized one takes 2+ minutes to boot and stutters during navigation.

---

## Part 3 — Running the App

Once the emulator is running, from the project root:

```bash
adb devices              # should list "emulator-5554  device"
npm run android          # builds and installs the app
```

Sanity check before first build: `adb devices` should list a device, and `./gradlew --version` from `android/` should show Gradle + JVM 17.

---

## Part 4 — Project Status Snapshot

### TL;DR

The `android/` project is **scaffolded and bootable** — RN 0.83, New Architecture on, Hermes on, package `com.oakplatforms.nanza`. But it has never been run end-to-end against the real app. Before it'll work end-to-end, we need to (1) wire up local Android tooling, (2) backfill Android-side configuration that today only exists on iOS (fonts, permissions, deep links, Stripe, camera), (3) replace the placeholder launcher icon and signing config, and (4) stand up a release pipeline (signed AAB + Play Console listing).

### Current Android State (what's already done)

| Area | Status | Notes |
|---|---|---|
| `android/` Gradle project | ✅ exists | RN 0.83 template, autolinking via `native_modules.gradle` |
| Package / namespace | ✅ `com.oakplatforms.nanza` | matches iOS bundle id |
| New Architecture | ✅ enabled | `newArchEnabled=true` in `gradle.properties` |
| Hermes | ✅ enabled | `hermesEnabled=true` |
| `MainActivity` / `MainApplication` | ✅ Kotlin, default RN template | uses `DefaultReactActivityDelegate` w/ Fabric flag |
| `AndroidManifest.xml` | ⚠️ minimal | only `INTERNET` permission, no deep-link intent filter |
| App icon | ❌ default RN icon | `mipmap-*/ic_launcher.png` are still the RN green-square placeholders |
| `strings.xml` | ✅ `app_name = Nanza` | |
| Signing | ❌ debug keystore only | `release` block currently re-uses `signingConfigs.debug` |
| ProGuard / R8 | ⚠️ disabled | `enableProguardInReleaseBuilds = false` |
| Architectures built | `armeabi-v7a, arm64-v8a, x86, x86_64` | fine for dev; for Play upload we ship an AAB and let Play split |

---

## Part 5 — Android Configuration Gaps (parity with iOS)

Several things are configured in `ios/nanza/Info.plist` but have no Android counterpart yet. These will silently break features at runtime.

### 5a. Fonts
- iOS lists ~30 custom font files in `UIAppFonts` (Figtree, Geist, EuclidCircularB, Volte).
- The `react-native.config.js` declares `assets: ['./assets/fonts/']`, so `npx react-native-asset` should copy them into `android/app/src/main/assets/fonts/` automatically.
- **Today:** `android/app/src/main/assets/fonts/` does not exist. Run `npx react-native-asset` (or the legacy `react-native link`) once after tooling is set up. Verify by checking that folder is populated and that text on Android renders in our brand fonts (not Roboto fallback).
- Note: `link-assets-manifest.json` already tracks 28 of the fonts but is missing the `Volte-*` and `Figtree-Black/BlackItalic` entries that iOS has — worth reconciling.

### 5b. Permissions (must be added to `AndroidManifest.xml`)
The app uses camera, photo picker, and location strings on iOS. Android equivalents:
```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />   <!-- Android 13+ -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
                 android:maxSdkVersion="32" />                              <!-- Android ≤12 -->
<uses-feature android:name="android.hardware.camera" android:required="false" />
```
Driven by `react-native-vision-camera` and `react-native-image-picker`. Vision Camera also needs Kotlin 1.9+ — we're on **1.8.0**, which will likely need bumping (see Part 7).

### 5c. Deep linking
iOS registers the `nanza://` URL scheme. Android needs an `<intent-filter>` on `MainActivity`:
```xml
<intent-filter android:autoVerify="false">
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="nanza" />
</intent-filter>
```
If we eventually want App Links (https URLs), we'll also need `assetlinks.json` hosted on the domain.

### 5d. Stripe
`@stripe/stripe-react-native` requires:
- `minSdkVersion >= 21` ✅ (we're at 21)
- App theme must inherit from `Theme.AppCompat` ✅ (we already use `Theme.AppCompat.DayNight.NoActionBar`)
- For Google Pay later: Play Services Wallet + a `meta-data` entry. Not blocking initial launch.

### 5e. App Transport Security equivalent
iOS has `NSAllowsArbitraryLoads=false` + `NSAllowsLocalNetworking=true`. Android default is `cleartextTraffic=false` from API 28+. If our dev/staging API uses HTTPS, no action needed. If we hit any cleartext endpoint in dev, we'll need a `network_security_config.xml`.

---

## Part 6 — Branding & Store Assets

### 6a. App icon
- Current: default RN green square in all `mipmap-*` density buckets.
- Need: adaptive icon (foreground + background layers) in `mipmap-anydpi-v26/ic_launcher.xml`, plus PNG fallbacks for older devices.
- Easiest: open the iOS `AppIcon.appiconset` master in Android Studio's **Image Asset Studio** (Resource Manager → New → Image Asset → Launcher Icons (Adaptive & Legacy)).

### 6b. Splash screen
- iOS uses `LaunchScreen.storyboard`. Android currently has no splash configured — it'll show a blank window on cold start.
- Recommended: add `react-native-bootsplash` (works on both platforms, single source asset, supports Android 12+ SplashScreen API). Lighter alternative: a styled `windowBackground` drawable, but then Android 12+ will overlay its own splash on top.

### 6c. Play Store listing assets
- 512×512 PNG hi-res icon
- 1024×500 feature graphic
- At least 2 phone screenshots (16:9 or 9:16, min 320 px)
- Optional: 7" + 10" tablet screenshots
- Short description (≤80 chars), full description (≤4000 chars)
- Privacy policy URL (required for any app that requests sensitive permissions — camera + photos qualifies)

---

## Part 7 — Release Build & Signing

1. **Generate an upload keystore** (one-time, store it in 1Password or similar — losing it means we can never update the app):
   ```bash
   keytool -genkeypair -v -storetype PKCS12 \
     -keystore nanza-upload.keystore \
     -alias nanza-upload -keyalg RSA -keysize 2048 -validity 10000
   ```
2. **Add credentials** to `~/.gradle/gradle.properties` (NOT checked into git):
   ```
   NANZA_UPLOAD_STORE_FILE=nanza-upload.keystore
   NANZA_UPLOAD_KEY_ALIAS=nanza-upload
   NANZA_UPLOAD_STORE_PASSWORD=...
   NANZA_UPLOAD_KEY_PASSWORD=...
   ```
3. **Wire `signingConfigs.release`** in `android/app/build.gradle` and point `buildTypes.release.signingConfig` at it (currently it points at `signingConfigs.debug` — a Play Console reject).
4. **Enable R8 minification** for release (`enableProguardInReleaseBuilds = true`) and keep an eye on the first release build for missing `-keep` rules (Stripe, Reanimated, Vision Camera all sometimes need them).
5. **Bump `versionCode` / `versionName`** before each upload (currently `1` / `"1.0"`).
6. **Build the AAB:**
   ```bash
   cd android && ./gradlew bundleRelease
   ```
   Output lands at `android/app/build/outputs/bundle/release/app-release.aab`.

Optionally enroll in **Play App Signing** (recommended) — Play holds the production signing key, we only manage the upload key.

---

## Part 8 — Library / Compatibility Risks

A few things in `package.json` that warrant a smoke test on Android specifically:

| Library | Android note |
|---|---|
| `react-native-vision-camera` ^4.7 | Needs Kotlin **1.9+** and `compileSdk 34`. We're on Kotlin **1.8.0** — almost certain to need a bump in `android/build.gradle` (`kotlinVersion = "1.9.24"`). |
| `react-native-reanimated` ^3.19 | Babel plugin already required; verify Hermes + New Arch combo works on Android — Reanimated 3 supports both but warm-up can be flaky. |
| `react-native-svg` ^15 | Fine on Android, but `react-native-svg-transformer` needs `metro.config.js` to apply on both platforms (verify). |
| `lottie-react-native` ^7.2 | Requires `androidx.appcompat` 1.6+ — should be auto-resolved. |
| `@aws-amplify/react-native` | Requires `react-native-get-random-values` (✅ have it) and `@react-native-async-storage/async-storage` (✅). |
| `react-native-fs` ^2.20 | Requests `READ_EXTERNAL_STORAGE` on older Android — make sure the manifest entry above is present. |
| `react-native-webview` ^13.16 | No Android-specific config needed beyond autolinking. |
| `flipper-integration` | Currently always linked. RN has been removing Flipper — fine for now, but if release builds choke, drop the `implementation("com.facebook.react:flipper-integration")` line and the `ReactNativeFlipper.initializeFlipper` call. |

---

## Part 9 — Suggested Sequence

1. **Local toolchain** (Parts 1–2) → confirm `./gradlew assembleDebug` succeeds.
2. **Bump Kotlin to 1.9.24**, run a debug build, fix any autolinking errors.
3. **Run `npx react-native-asset`** to copy fonts.
4. **Add manifest permissions + deep-link intent filter** (5b, 5c).
5. **`npm run android`** on emulator → walk through camera flow, image picker, Stripe checkout, deep link (`adb shell am start -W -a android.intent.action.VIEW -d "nanza://..." com.oakplatforms.nanza`).
6. **Replace launcher icon** + add splash (6a, 6b).
7. **Generate upload keystore + wire release signing** (Part 7).
8. **First `bundleRelease`** → upload to Play Console **Internal Testing** track.
9. **Privacy policy + store listing assets** → promote to Closed/Open testing → Production.

---

## Part 10 — Open Questions

- Do we want to use **EAS Build / a managed CI** (GitHub Actions, Bitrise, Codemagic) for Android builds, or just local `./gradlew bundleRelease` for now?
- Do we need **Firebase** on Android (FCM for push, Crashlytics)? Neither is wired today on either platform.
- Which Google Play Console org / developer account hosts Nanza?
- Minimum supported Android version — keep at **API 21 (Android 5.0)** or raise to API 24/26 to drop legacy code paths?
