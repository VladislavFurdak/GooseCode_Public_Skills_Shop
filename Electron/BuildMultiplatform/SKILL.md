---
name: electron-build-multiplatform
description: Packaging an Electron app for every shipping target with electron-builder — macOS dmg/zip for Apple Silicon AND Intel with hardened runtime + entitlements, Linux AppImage + deb built from a Windows machine via Docker, Windows portable/nsis. Includes the which-OS-can-build-what matrix and a working config lifted from a production app (GooseCode). Use at first-release time, and read the WASM-over-native advice earlier than that.
---

# One codebase, five artifacts, three operating systems

Target list for a serious desktop release, and where each can be built:

| Artifact | Arch | Can be built on |
|---|---|---|
| Windows `nsis` installer or `portable` exe | x64 | Windows |
| macOS `dmg` + `zip` | **arm64 (Apple Silicon)** | **macOS only** |
| macOS `dmg` + `zip` | **x64 (Intel)** | **macOS only** |
| Linux `AppImage` | x64 | Linux, or **Docker from Windows** (§4) |
| Linux `deb` | x64 | Linux, or **Docker from Windows** (§4) |

Two facts drive the whole strategy. **macOS artifacts require macOS** —
signing, DMG creation and notarization need Apple's toolchain; no Docker
route exists. From a Windows machine you produce Windows + Linux yourself
and run the mac build on a Mac (or mac CI runner) — tell the user this
early, not at release day. **Everything must be bundled before packaging**
— electron-builder packages `dist/`, it does not compile; every `dist:*`
script below chains the app build first.

## 1. Order of operations and the files allowlist

Working reference (versions actually shipped: Electron 38,
electron-builder 26):

```jsonc
// package.json
"scripts": {
  "dist:win":      "npm run build && electron-builder --win --publish never",
  "dist:linux":    "npm run build && electron-builder --linux --publish never",
  "dist:appimage": "npm run build && electron-builder --linux AppImage --publish never",
  "dist:deb":      "npm run build && electron-builder --linux deb --publish never",
  "dist:mac":      "npm run build && electron-builder --mac --publish never"
},
"build": {
  "appId": "com.example.myapp",
  "productName": "My App",
  "directories": { "output": "release" },
  "files": ["dist/**/*", "package.json"],
  "asar": true,
  "npmRebuild": false,
  "protocols": [{ "name": "MyApp", "schemes": ["myapp"] }]
}
```

- `files` is an **allowlist** — `dist/**` plus `package.json` and nothing
  else. Without it, electron-builder ships your entire `node_modules` and
  sources; a 100 MB app becomes 600 MB and includes your `.env` if one
  exists. The bundler already put everything needed into `dist/`.
- `npmRebuild: false` is correct **only** while no runtime native modules
  exist — which §5 argues to make true on purpose.
- `asar: true` unless the app spawns bundled binaries/loads loose files at
  runtime — then either `asar: false` (the reference app's choice: it ships
  ripgrep, WASM grammars, fonts as real files) or `asarUnpack` for just
  those paths. A packaged app that worked in dev and dies with `ENOENT`
  inside `app.asar` needs this paragraph.
- `--publish never` keeps electron-builder from trying to upload releases
  because a `GH_TOKEN` happened to exist in the environment.
- `protocols` makes installers register the deep-link scheme
  (`electron-supabase-setup` §3 depends on it).

Give every `dist:*` invocation **at least 10 minutes** (`timeout_ms:
600000`): first runs download Electron per-platform-per-arch plus
platform tooling, and a killed download poisons the cache with a
half-file whose next error mentions checksums, not timeouts.

## 2. macOS — both silicons, hardened runtime, the ASCII plist

```jsonc
"mac": {
  "target": [
    { "target": "dmg", "arch": ["arm64", "x64"] },
    { "target": "zip", "arch": ["arm64", "x64"] }
  ],
  "category": "public.app-category.developer-tools",
  "artifactName": "MyApp-mac-${arch}.${ext}",
  "hardenedRuntime": true,
  "entitlements": "build/entitlements.mac.plist",
  "entitlementsInherit": "build/entitlements.mac.plist",
  "gatekeeperAssess": false,
  "notarize": false,
  "darkModeSupport": true
}
```

- **Per-arch beats universal** as the default: `arch: ["arm64","x64"]`
  yields four artifacts users pick from; a `universal` build is one
  download at nearly double size. Ship per-arch unless distribution can't
  ask the user which Mac they have.
- `zip` targets are not optional decoration — auto-update on macOS
  (electron-updater) updates from the zip, dmg is for humans.
- `notarize: false` for unsigned/dev cycles, with a separate signed script
  flipping it: `electron-builder --mac -c.mac.notarize=true` (the reference
  app keeps exactly this pair). Signing+notarization needs the user's Apple
  Developer ID cert and takes Apple-side minutes; unsigned mac builds still
  run via right-click → Open, or `xattr -cr "My App.app"` — document that
  for testers instead of pretending unsigned is undistributable.

`build/entitlements.mac.plist` — these four, each load-bearing, taken
verbatim from production:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- V8's JIT: Electron will not start under hardened runtime without these two. -->
  <key>com.apple.security.cs.allow-jit</key><true/>
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key><true/>
  <!-- Only if the app runs binaries signed by a different identity (bundled
       ripgrep, native helpers). Delete if it doesn't. -->
  <key>com.apple.security.cs.disable-library-validation</key><true/>
  <!-- Only if spawned children must inherit env (PATH, proxies). Delete if not. -->
  <key>com.apple.security.cs.allow-dyld-environment-variables</key><true/>
</dict>
</plist>
```

**Keep this file ASCII-only.** codesign's plist reader rejects any non-ASCII
byte — one em-dash or smart quote in a comment and signing fails with
`cannot read entitlement data`, an error that sends you everywhere except
the comment you just wrote. (Production-verified the hard way.)

## 3. Windows and Linux targets

```jsonc
"win": {
  "target": [{ "target": "nsis", "arch": ["x64"] }],   // or "portable"
  "icon": "build/icon.ico"                              // real .ico with a 256px layer
},
"nsis": { "oneClick": true, "artifactName": "MyApp-Setup-${version}.${ext}" },
"linux": {
  "target": [
    { "target": "AppImage", "arch": ["x64"] },
    { "target": "deb", "arch": ["x64"] }
  ],
  "icon": "build/icon.png",
  "executableName": "my-app",
  "synopsis": "One-line description",
  "category": "Development",
  "maintainer": "You <you@example.com>",
  "artifactName": "MyApp-${version}.${ext}"
}
```

- `portable` (single .exe, no install) suits tools; `nsis` suits apps that
  need shortcuts/uninstall/protocol registration at install time. Unsigned
  Windows builds trip SmartScreen ("unrecognized app") — reputation or an
  EV/OV cert fixes it; say so, don't chase it as a bug.
- `maintainer` is **required** for deb (dpkg metadata) — its absence is a
  build error discovered only on the Linux pass. `executableName` pins the
  binary/`.desktop` name; `category`/`synopsis` feed desktop menus.
- AppImage on current Ubuntu/Debian needs `libfuse2`, which no longer ships
  by default. Not your bug, but your support ticket: document
  `sudo apt install libfuse2` or the no-fuse fallback
  `./MyApp.AppImage --appimage-extract-and-run`. The deb exists precisely
  so apt users never meet this.

## 4. Linux artifacts from a Windows machine — Docker

Production-verified pattern (the reference app's
`scripts/docker-build-linux.sh`), PowerShell invocation:

```powershell
docker run --rm -v "C:\path\to\app:/project" `
  -v myapp-linux-nm:/project/node_modules `
  -v myapp-electron-cache:/root/.cache/electron `
  -w /project electronuserland/builder:latest `
  bash -lc "npm ci && npm run dist:linux"
```

The **named volume over `node_modules` is the load-bearing trick** — the
same platform-contamination law as `NextJs/NextJsDockerCompose` §5, in the
opposite direction: the bind mount would expose your *Windows*
`node_modules` (win32 binaries, `electron.exe`) to a Linux build, failing
in creatively misleading ways. The named volume shadows it with a
container-local dir where `npm ci` installs Linux deps; the electron-cache
volume keeps the ~100 MB Electron download from re-running every build.
Artifacts land in `release/` on the Windows side via the bind mount, named
volumes persist between runs.

If the app build runs on Bun (as the reference does), install Bun inside
the container first — `electronuserland/builder` ships node/npm/yarn only.

## 5. The strategic decision that makes all of this easy: avoid native modules

Every runtime native module multiplies packaging: rebuilds per Electron ABI
× per platform × per arch (`npmRebuild`/`@electron/rebuild`), asar unpacking,
mac library-validation entitlements. The reference app packages for five
targets from two OSes with `npmRebuild: false` because it chose **WASM and
pure-JS at every fork**: tree-sitter grammars as `.wasm`, pdfium as WASM,
OCR via tesseract.js — one artifact runs on every platform+arch unchanged.

When you need capability that looks native (grep, parsing, PDF, SQLite):
first look for the WASM build or a prebuilt-static-binary-per-platform you
spawn (ripgrep pattern); accept a true native module only when benchmarks
say you must, and budget the packaging cost at that moment, not at release.

## 6. Auto-update, briefly

electron-updater (nsis + AppImage + mac zip targets are its food) with a
generic HTTPS or GitHub provider; the reference app wraps it in an
`UpdaterManager` driving renderer events (`update:status/download/install`
channels — `electron-ipc-contract` shapes). Two rules: never auto-install
without consent (download → ask → `quitAndInstall`), and mac auto-update
**requires signed builds** — unsigned mac distributions update by "download
the new dmg yourself", design the UX for that honestly.

## Verify — per artifact, not per build log

A green electron-builder run proves packaging, not a working app. Launch
each artifact on its real platform (VMs count; WSLg runs AppImages):

1. Windows exe/installer: installs, launches, uninstalls clean; protocol
   link (`start myapp://test`) reaches the running app.
2. AppImage on stock Ubuntu: launches (or fails with the documented fuse
   message, not a mystery); deb: `sudo apt install ./MyApp.deb` then launch
   from the desktop menu — icon, name and category correct.
3. mac arm64 AND x64 artifacts each launch on matching hardware; signed
   builds pass `spctl -a -vv "My App.app"`.
4. Every artifact: the §1 allowlist held — installer size is dist-sized,
   not node_modules-sized; no `.env`, no sourcemaps unless intended.
5. Packaged-app smoke: deep link, file dialogs, and the §Verify list of
   `electron-supabase-setup` if wired — several bugs (asar paths, protocol
  registration) exist **only** packaged.

## Notes

- Version pins matter here more than anywhere: electron-builder majors
  change signing/notarization behavior. The reference shipped on
  electron-builder 26.15 / Electron 38 — record your pair in the repo when
  you first get all five artifacts green.
- CI mapping of §"can be built on": one Windows/Linux runner pair + one
  macOS runner covers the matrix; the Docker trick collapses Windows+Linux
  onto one machine for local releases.
- Related: `electron-create-app` (emits the `dist/` this packages),
  `electron-supabase-setup` (protocols + baked env),
  `NextJs/NextJsDockerCompose` §5 (the same node_modules law, web edition),
  `rn-build-ios-android` (the mobile sibling of the platform matrix).
