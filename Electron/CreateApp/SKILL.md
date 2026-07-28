---
name: electron-create-app
description: Scaffolds an Electron + React + TypeScript desktop app with the three-process model and the security settings that must never be loosened — contextIsolation, no nodeIntegration, sandbox on, CSP with no CDN. Use BEFORE any desktop app work when no project exists. Patterns cross-checked against a production Electron app (GooseCode - Electron 38, sandboxed preload, Bun-bundled).
---

# Electron scaffold: three processes, locked down from commit one

An Electron app is three programs with three different capability sets:

| Process | Runs | May do | Built as |
|---|---|---|---|
| **main** | Node.js | everything — fs, spawn, windows, native APIs | ESM |
| **preload** | isolated bridge context | expose a chosen API to the renderer | **CJS** (see below) |
| **renderer** | Chromium, no Node | DOM + whatever preload exposed | browser ESM |

Every architectural decision in the other Electron skills flows from this
table: privileged work lives in main, the renderer is treated as untrusted,
and the preload is the only door between them (`electron-ipc-contract`).

## 1. Scaffold

electron-vite is the maintained scaffold for this stack:

```bash
npm create @quick-start/electron@latest <app-name> -- --template react-ts
cd <app-name> && npm i     # timeout: 300000 — downloads Electron itself (~100 MB)
```

The Electron binary download is the slow step; killing it mid-way leaves a
broken `node_modules/electron` whose error (`Electron failed to install
correctly`) points nowhere near the cause. Give installs 5 minutes. (Corporate
networks that block the download need the `ELECTRON_MIRROR` env var — say so
rather than retrying.)

You get `src/main/index.ts`, `src/preload/index.ts`, `src/renderer/` with
React, and `electron.vite.config.ts` wiring HMR for the renderer plus rebuilds
for main/preload. Add a `src/shared/` folder yourself immediately — IPC
channel names and types live there (`electron-ipc-contract`) and it must stay
dependency-free so all three processes can import it.

An alternative that this shop's reference app (GooseCode) uses: no vite, a
~300-line `Bun.build` script bundling the three entrypoints separately. It
buys full control (feature-flag `define` tables, content-hashed asset names)
at the cost of owning the pipeline. Default to electron-vite; copy the Bun
pipeline only if the project already runs on Bun.

## 2. The window — security settings are not preferences

The production-verified shape:

```ts
const win = new BrowserWindow({
  width: 1280, height: 800,
  minWidth: 960, minHeight: 600,
  backgroundColor: "#ffffff",       // pre-paint color — kills the white/black flash; theme-matched in electron-modern-design
  autoHideMenuBar: true,            // Windows/Linux; macOS menu handling is in electron-modern-design
  webPreferences: {
    preload: PRELOAD_PATH,
    contextIsolation: true,         // renderer JS cannot reach preload internals
    nodeIntegration: false,         // renderer has no require/process
    sandbox: true,                  // renderer + preload run in the OS sandbox
  },
});
```

These three `webPreferences` are the entire security model of the app.
A renderer that renders any remote or user-influenced content (a link
preview, an embedded page, markdown with HTML) with `nodeIntegration: true`
is remote code execution on the user's machine. There is no feature that
justifies flipping them — whatever needed Node in the renderer belongs in
main, behind IPC. Treat any "just set sandbox: false" advice found in old
Stack Overflow answers as wrong by default.

**`sandbox: true` changes what the preload may be.** A sandboxed preload is
loaded as plain CommonJS and can `require` only `electron` and Node built-ins
polyfilled by Electron — no ESM, no arbitrary npm imports at runtime.
Consequences:

- The preload must be **emitted as CJS** even in a `"type": "module"` project.
  electron-vite does this when the file is named `index.ts` under
  `src/preload` (output `.mjs` must be avoided — check `out/preload/` ends in
  `.js`/`.cjs`). GooseCode's Bun pipeline sets `format: "cjs"`,
  `naming: "[dir]/[name].cjs"` explicitly for exactly this reason.
- Any npm library the preload needs must be **bundled into** it, not required
  at runtime. Keep the preload tiny; logic belongs in main anyway.

Two more window-level rules from production:

```ts
// Links open in the user's browser, never in a new Electron window.
win.webContents.setWindowOpenHandler(({ url }) => {
  if (url.startsWith("https://")) shell.openExternal(url);
  return { action: "deny" };
});
```

```ts
// One instance. Without this, second launches spawn a second app; with it,
// they focus the first — and deliver deep links (electron-supabase-setup).
if (!app.requestSingleInstanceLock()) app.quit();
app.on("second-instance", () => {
  if (win) { if (win.isMinimized()) win.restore(); win.focus(); }
});
```

## 3. CSP — no CDN, ever

In the renderer's `index.html`:

```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' https:">
```

A desktop app must work offline and must not execute remote script. That
outlaws the web habit of CDN `<script>` tags and Google-Fonts `<link>`s —
fonts and icons are vendored into the bundle instead
(`electron-modern-design` §2 has the mechanics). `connect-src https:` stays
only if the app talks to an API (e.g. Supabase); tighten it to the actual
origin at release. Set the policy now: adding CSP after screens exist turns
into whack-a-mole with every inline handler that snuck in.

## 4. Dev loop and scripts

```bash
npm run dev        # electron-vite dev — renderer HMR, main/preload rebuild + relaunch
npm run typecheck  # add if the template lacks it: tsc --noEmit for both node and web tsconfigs
```

Renderer edits hot-reload; **main and preload edits restart the app** — if a
main-process change seems to "not apply", the dev process wasn't watching it;
restart `npm run dev` before debugging the code.

The template's `npm run build` produces `out/` (or `dist/`) with the three
bundles. Packaging that output into installers is deliberately NOT here — that
is `electron-build-multiplatform`, and it only packages what this step emits.

## 5. Verify before handing over

1. `npm run dev` opens the window; devtools console has zero errors **and
   zero CSP violations**.
2. `npm run typecheck` passes.
3. In devtools: `window.require` is `undefined`, `process` is `undefined`
   (isolation is real), and the API surface from preload exists once
   `electron-ipc-contract` is wired.
4. An external `https:` link in the app opens the system browser, not a new
   Electron window.
5. Launching the app twice focuses the first instance.

## What not to do here

- Do not build UI yet — `electron-ipc-contract` (the main↔renderer contract)
  and `electron-modern-design` (tokens, fonts, titlebar) come first, same
  ordering logic as scaffold-then-shadcn on the web side.
- Do not add `electron-store`, a DB, or settings code yet —
  `electron-state-management` owns where state lives.
- Do not `loadURL` anything remote in production. The renderer loads the
  bundled `index.html`, full stop.
- Do not disable `webSecurity`, add `allowRunningInsecureContent`, or register
  a catch-all `file://` handler to "fix" an asset 404 — fix the asset path.

## Notes

- Electron major versions move fast (Chromium cadence). The reference app pins
  `electron` in devDependencies and upgrades deliberately, not via `latest`.
- `dialog`, `shell`, `clipboard`, `Menu` — all main-process; the renderer asks
  for them over IPC. If a renderer file imports from `"electron"`, the
  architecture has already gone wrong.
- Related: `electron-ipc-contract` (next step), `electron-modern-design`,
  `electron-state-management`, `electron-build-multiplatform` (packaging),
  `electron-supabase-setup` (uses the single-instance + protocol hooks from
  here).
