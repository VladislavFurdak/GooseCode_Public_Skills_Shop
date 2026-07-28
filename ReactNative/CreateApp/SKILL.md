---
name: expo-create-app
description: Scaffolds a React Native app with Expo and expo-router — the framework the RN docs themselves recommend. Use BEFORE any mobile app work when no project exists yet. Explains the one decision that shapes everything after (Expo Go vs development build) and what a Windows machine can and cannot run. Not for an app that already exists.
---

# React Native scaffold: Expo, and why

Bare React Native (`react-native init`) is deprecated as a starting point — the
official RN docs send new projects to a framework, and that framework is Expo.
Choosing bare means hand-rolling routing, native config management, builds and
updates that Expo ships working. Every other skill in this folder assumes Expo.

**Expo is not "Expo Go".** Expo is the framework and build toolchain; Expo Go is
a demo sandbox app. Conflating them is the #1 source of confusion — see §3.

## 1. Scaffold

Run from the directory that should CONTAIN the project:

```bash
npx create-expo-app@latest <app-name> --yes
```

**Give it at least 5 minutes** (`timeout_ms: 300000`, or leave unset). It
installs the full dependency tree; killing it mid-install leaves a half-created
directory that fails retry with a conflict error — the same failure mode as
`create-next-app`, and just as annoying to clean up.

The default template already includes TypeScript, `expo-router` (file-based
routing), React Navigation under the hood, `react-native-reanimated`,
`react-native-safe-area-context` and example screens. Do not add a router or
navigation library — one is already there.

The template ships demo screens. Clear them before building:

```bash
npm run reset-project
```

This moves the examples to `app-example/` (delete it) and leaves a minimal
`app/` with a blank `index.tsx` and `_layout.tsx`. Build on that, not around
the demo code.

## 2. What the scaffold gives you

```
app/
  _layout.tsx      root layout — providers, theme, auth gate live here
  index.tsx        the "/" screen
app.json           app config: name, slug, scheme, icons, plugins
```

Routing is the filesystem: `app/settings.tsx` is `/settings`,
`app/(tabs)/_layout.tsx` declares a tab navigator, `app/user/[id].tsx` is a
dynamic route. Navigate with `<Link href="/settings">` or
`router.push("/settings")` from `expo-router`.

Set these in `app.json` NOW, not at release time:

```jsonc
{
  "expo": {
    "name": "My App",
    "slug": "my-app",
    "scheme": "myapp",                          // deep-link scheme — Supabase OAuth needs it
    "ios": { "bundleIdentifier": "com.you.myapp" },
    "android": { "package": "com.you.myapp" }
  }
}
```

`bundleIdentifier` / `package` are **immutable once submitted to a store**.
`scheme` is required by the auth flow in `rn-supabase-setup`; adding it later
forces a rebuild of the native app.

The New Architecture is the default on current SDKs — do not set
`newArchEnabled: false` to "fix" a library problem; the libraries in these
skills all support it, and the old architecture is on its way out.

## 3. Expo Go vs development build — the decision that shapes everything

| | Expo Go | Development build |
|---|---|---|
| What it is | Prebuilt sandbox app from the store | YOUR app with dev tooling, built by EAS |
| Native modules | Only the fixed set baked into Expo Go | Anything |
| Get it running | Scan a QR code, 30 seconds | One `eas build` (~15 min, then cached) |
| Good for | First screens, pure-JS work | Everything after |

**Start in Expo Go.** The moment you add a native module outside Expo Go's
built-in set — `react-native-mmkv`, most non-Expo libraries — the app crashes in
Expo Go with `Cannot find native module ...`. That error does not mean the
library is broken; it means you have outgrown Expo Go. Build a development
client (`rn-build-ios-android` §3) and keep the same dev loop.

Do not "fix" it by hunting for a JS-only alternative to a good native library —
switching to a dev build is the intended path, not a failure.

## 4. Run it

```bash
npx expo start
```

- `a` — launch on Android emulator (needs Android Studio + an AVD)
- `w` — run in the browser (react-native-web; useful smoke test, not the product)
- QR code — scan with the Expo Go app on a physical device (same Wi-Fi network)

**On Windows there is no iOS simulator.** The realistic Windows matrix is:
Android emulator locally + a physical iPhone running Expo Go (later: a
development build from EAS). Do not tell the user to press `i` on Windows — it
fails. macOS is only required for *local* iOS builds; EAS cloud builds remove
even that (see `rn-build-ios-android`).

If a physical device cannot connect (corporate Wi-Fi, VPN):
`npx expo start --tunnel`.

## 5. Verify before handing over

```bash
npx tsc --noEmit
npx expo-doctor
```

`expo-doctor` cross-checks every installed package against the SDK's expected
versions — the RN ecosystem is version-locked to a degree web developers do not
expect, and a mismatched package produces runtime crashes with useless stack
traces. Fix what it reports with:

```bash
npx expo install --fix
```

**Always add RN packages with `npx expo install <pkg>`, never `npm i <pkg>`.**
`expo install` resolves the version that matches the current SDK; `npm i` grabs
latest, which regularly targets a newer RN than the project has.

## What not to do here

- **Do not run `npx expo prebuild` / eject.** The `android/` and `ios/` folders
  it generates put native config under manual management permanently. Config
  plugins in `app.json` cover the cases that seem to need it.
- **Do not mix package managers.** One lockfile. The scaffold used npm; stay
  with npm.
- **Do not install `react-native-cli` or follow "React Native CLI" tutorials** —
  they describe the bare workflow this skill deliberately avoids.
- **Do not start building screens yet** — wire up design first
  (`rn-modern-design`), the same way the Next.js skills scaffold before shadcn.

## Hand-off

Ready when `npx expo start` shows the blank index screen on at least one real
target (emulator or device) and `expo-doctor` is clean. Next:
`rn-modern-design` (UI foundation), then `rn-state-management`,
`rn-supabase-setup`, `rn-localization` as the app grows, and
`rn-build-ios-android` when it needs to leave the dev loop.
