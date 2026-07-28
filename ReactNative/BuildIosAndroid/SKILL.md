---
name: rn-build-ios-android
description: Building and shipping an Expo app for iOS and Android with EAS Build — profiles, credentials, what a Windows machine can produce (everything, via the cloud — including iOS), development builds, store submission and OTA updates. Use when the app first needs to run outside Expo Go, and again at release time. Names the two accounts only the user can create.
---

# iOS and Android builds without a Mac requirement

Native iOS builds require macOS + Xcode — locally. **EAS Build removes the
"locally":** it builds both platforms on Expo's cloud machines, so a Windows
machine ships iOS apps. That single fact decides the whole strategy here:
cloud builds are the default path, local builds the optimization.

The three build kinds, in the order a project meets them:

| Kind | What for | Profile |
|---|---|---|
| Development build | Your app + dev tooling — replaces Expo Go the moment a native module appears | `development` |
| Preview | Installable release-mode binary for testers (APK / simulator app / ad-hoc iOS) | `preview` |
| Production | Store-ready AAB / IPA | `production` |

## 1. One-time setup

```bash
npm i -g eas-cli
eas login          # needs an Expo account — free tier includes cloud builds with a queue
eas build:configure   # writes eas.json, links the project (creates projectId in app.json)
```

`eas login` is interactive and needs the **user's** Expo account — like the
hosted-Supabase rule in `NextJs/SupabaseSetup` §7: ask, don't fake it. Two more
accounts appear later and are also strictly the user's to create: an **Apple
Developer Program** membership ($99/yr — required for iOS on-device builds,
TestFlight, App Store) and a **Google Play Console** account ($25 once). Until
those exist you can still ship: Android APKs install directly, iOS simulator
builds need no Apple account at all.

A serviceable `eas.json`:

```jsonc
{
  "cli": { "appVersionSource": "remote" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": { "simulator": true }        // flip to false when building for a real iPhone
    },
    "preview": {
      "distribution": "internal",
      "android": { "buildType": "apk" }   // APK installs directly; default AAB does NOT
    },
    "production": { "autoIncrement": true }
  },
  "submit": { "production": {} }
}
```

`appVersionSource: remote` + `autoIncrement` hands build-number bumping to EAS
— versionCode/buildNumber collisions at submission simply stop existing.
Marketing version stays yours in `app.json` (`"version": "1.0.0"`).

## 2. Identity and assets — before the first build

From `expo-create-app`: `ios.bundleIdentifier` and `android.package` are
frozen at first store submission. Also in `app.json`:

- `icon` — 1024×1024 PNG, no transparency for iOS.
- `android.adaptiveIcon` — `foregroundImage` (transparent PNG, content within
  the middle ~66% safe zone) + `backgroundColor`. Skipping this puts your
  full-bleed icon inside a white circle on most launchers.
- Splash via the `expo-splash-screen` plugin entry.

Run `npx expo-doctor` before every first-of-a-kind build — it catches the
version mismatches that otherwise surface as cloud-build failures after a
20-minute queue.

## 3. Building

```bash
eas build --platform android --profile development
eas build --platform ios --profile development
eas build --platform all --profile production
```

Run these **in the background / detached** — a cloud build is queue + ~10–25
minutes, far beyond any tool timeout. `eas build:list` (or the printed build
URL) reports status; the artifact is a download link when done.

First-build credential prompts — answer yes and move on:

- **Android keystore**: let EAS generate and store it. The keystore signs
  every future update of the app; losing a self-managed one means never
  updating the app again. EAS-managed is the safe default, and
  `eas credentials` exports a backup.
- **iOS**: EAS drives certificates/profiles through the Apple Developer
  account (interactive Apple login on first run — the user again).
  Simulator builds skip all of it.

Installing the artifacts:

- Android APK → download to the device and open, or
  `adb install build.apk` on the emulator.
- iOS simulator build (macOS only) → unzip, drag the `.app` onto the simulator.
- iOS device (`"simulator": false`) → the device's UDID must be registered:
  `eas device:create` prints a link the user opens **on the iPhone** to
  enroll it, then rebuild. Skipping this is the classic "build succeeds,
  install fails" on iOS internal distribution.

A development build installed once replaces Expo Go: `npx expo start` and the
same QR/reload loop, now with every native module available. Rebuild only when
native modules or app config change — JS changes never need a rebuild.

Local alternative (`eas build --local`): Android works on Linux/macOS/WSL; iOS
still needs macOS. `npx expo run:android` with Android Studio installed is the
fastest all-local Android loop on Windows — but cloud builds keep working when
the local SDK setup doesn't, which is why they are the default in this skill.

## 4. Submission

```bash
eas submit --platform ios        # to App Store Connect / TestFlight
eas submit --platform android    # to Google Play
```

iOS submission is fully automatable once the Apple account exists. Google Play
has a hard first-time exception: **the very first AAB must be uploaded by hand
in the Play Console** (creating the app listing) before `eas submit` works for
that package — tell the user, hand them the AAB, continue from build two.
Store listings, screenshots and review answers are user work; the builds are
yours.

## 5. OTA updates — ship JS without the store

```bash
eas update:configure                                  # adds expo-updates + channels
eas update --channel production --message "fix: …"    # pushes the current JS bundle
```

Builds carry a **channel** (wired to the profile) and a **runtime version**;
an update applies only where both match. The rule that keeps OTA safe:
JS/asset-only changes are updatable; anything touching native code or config
(new native module, plugin change, SDK upgrade) changes the runtime version
and **requires a store build**. With `"runtimeVersion": { "policy":
"appVersion" }` in `app.json`, bumping `version` fences updates from older
binaries automatically — set it once and the footgun is gone.

## Verify

1. `eas build:list` shows the build finished (never assume from a started
   command — check).
2. Preview APK installs and runs on a physical Android device — release mode
   catches crashes dev mode hides (missing env vars, ProGuard-adjacent
   issues, `EXPO_PUBLIC_*` still pointing at a LAN IP — see
   `rn-supabase-setup` Notes).
3. iOS: simulator build boots, or TestFlight build reaches the device.
4. After an `eas update`: force-quit the installed app twice (updates apply on
   the *next* launch after download) and confirm the change arrived.

## Notes

- Free-tier cloud builds queue behind paid ones — minutes to tens of minutes.
  Plan around it; don't burn builds on what Expo Go / a dev client already
  answers.
- Every `eas build` consumes the git-committed *and* working-tree state as
  configured — but env vars come from `eas.json`/EAS secrets
  (`eas env:create`), NOT your local `.env`. Local-only secrets are the
  standard "works locally, crashes in TestFlight" cause.
- `npx expo prebuild` still isn't needed for any of this — EAS runs it
  internally per build and throws the result away. Keep the project managed.
- Related: `expo-create-app` §3 (when the dev build becomes mandatory),
  `rn-supabase-setup` (env per environment), `electron-build-multiplatform`
  (the desktop counterpart — same "what can this OS build" table idea).
