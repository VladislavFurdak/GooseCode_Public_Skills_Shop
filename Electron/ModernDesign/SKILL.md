---
name: electron-modern-design
description: Making an Electron renderer look like a desktop app, not a website in a frame — Tailwind tokens with dark mode wired to the OS, vendored fonts (no CDN, offline-first), per-platform titlebar and the macOS menu that Cmd+C silently depends on, and the small CSS rules (user-select, cursors, scrollbars) that separate native-feeling from webby. Use after the scaffold, before building screens.
---

# Desktop UI: a browser engine pretending it isn't one

The renderer is Chromium, so the web half of this shop applies directly —
Tailwind, tokens, the discipline of `nextjs-setup-shadcn` #4 (semantic colors,
never raw palette). This skill covers what the web skills cannot: the parts
where a desktop app must behave like a desktop app or users feel the fake.

## 1. Tailwind + tokens

Tailwind v4 works unmodified in an Electron renderer (the reference app runs
it via `@tailwindcss/cli`; with electron-vite use `@tailwindcss/vite`). Define
the same semantic token set as the web skills (`--background`, `--foreground`,
`--card`, `--border`, `--primary`…) in `:root` / `.dark` blocks, and hold
screens to tokens with the same grep audit. shadcn/ui itself works in Electron
renderers — it is just React + Tailwind; if the app wants that look, follow
`NextJs/SetupShadCN`'s contracts, minus the Next-specific hydration items.

**Tailwind's on-switch is the CSS entry file — and forgetting it fails
silently.** The entry the renderer imports must contain `@import "tailwindcss";`
(v4) or the three `@tailwind base/components/utilities;` directives (v3
projects; v3 directives inside a v4 pipeline also no-op). Miss it — or point
v3 `content` globs away from `src/renderer/**` — and the build still
succeeds: custom properties and resets load, **zero utility classes** are
generated, and the app renders as unstyled HTML while everyone debugs the
React components (which are fine). The tell is output size: a utilities-bearing
build emits tens of KB of CSS; a file under ~2 KB means Tailwind never ran.
`grep -c "flex\|rounded" out/renderer/assets/*.css` returning 0 confirms it.

Density is the desktop difference: base font 13–14px (not 16), 32px control
heights (not 44px web-mobile buttons), tighter spacing scale. A desktop tool
drawn at marketing-site scale reads as a toy.

## 2. Fonts and icons — vendored, never fetched

`electron-create-app` §3's CSP already blocks CDNs; this is the constructive
half. Ship the fonts inside the app:

```bash
npm i -D @fontsource/inter
```

Then copy (in the build pipeline, or import via the bundler) the CSS + woff2
files for exactly the weights used — the reference app vendors Inter
400/500/600 and Font Awesome's css+webfonts into
`dist/renderer/vendor/…` and links them relatively. What this buys:

- Works offline — a desktop app that renders fallback fonts without network
  looks broken in exactly the airport-wifi situations desktop apps get used.
- No FOUT on every cold start, no third-party request leaking usage.

Subset discipline: three weights of one family. Every extra weight is ~50 KB
in the installer forever. For icons prefer `lucide-react` (tree-shaken SVG
components, same set as the shop's web side) over icon fonts for new code.

## 3. Dark mode — three things must agree

1. **The OS signal.** `prefers-color-scheme` works in the renderer and tracks
   the OS (Electron feeds it from `nativeTheme`). For a user override
   (light/dark/system), store the choice in settings
   (`electron-state-management`) and stamp `document.documentElement.classList`
   — same class strategy as the web skills.
2. **The window pre-paint color.** `BrowserWindow`'s `backgroundColor` paints
   before the renderer loads. If it stays `#ffffff` while the app is dark,
   every launch and every new window flashes white — the single most common
   "cheap Electron app" tell. Read the persisted theme in main *before*
   constructing the window and pass the matching color.
3. **`nativeTheme.themeSource`.** Set it (`"light" | "dark" | "system"`) when
   the override changes so native surfaces — context menus, dialogs, scrollbar
   defaults on mac — flip with the app instead of staying OS-colored.

An app where those three disagree shows its seams at every launch and every
right-click.

## 4. Titlebar and menu — the per-platform contract

**Default frame first.** The OS titlebar with `autoHideMenuBar: true`
(Win/Linux) is a perfectly professional v1; custom titlebars are polish, not
foundation.

**The macOS menu is not optional.** On macOS, ⌘C/⌘V/⌘X/⌘A dispatch through
the application menu's Edit roles. A packaged app with no menu — common,
because Windows-based devs never see it — ships with **copy/paste silently
dead in every text field** on macOS. Register standard roles at startup:

```ts
if (process.platform === "darwin") {
  Menu.setApplicationMenu(Menu.buildFromTemplate([
    { role: "appMenu" }, { role: "fileMenu" }, { role: "editMenu" },
    { role: "viewMenu" }, { role: "windowMenu" },
  ]));
}
```

Accelerators use `CmdOrCtrl+…` so shortcuts land on ⌘ on mac and Ctrl
elsewhere. Rebuilding this menu on locale change is `electron-localization`
§4's job.

**Going frameless** (when the design calls for it):

- mac: `titleBarStyle: "hiddenInset"` keeps native traffic lights over your
  UI (`trafficLightPosition` to fine-tune) — do not draw fake mac buttons.
- Windows: `titleBarStyle: "hidden"` + `titleBarOverlay: { color, symbolColor,
  height }` keeps native min/max/close — do not draw fake ones; snap layouts
  and Fitts's-law edge behavior come free only with the real controls.
- Your bar becomes the drag region: `-webkit-app-region: drag` on it,
  `no-drag` on every interactive child (miss one button and it becomes
  an undraggable dead zone that *moves the window* when clicked).
- Reserve the traffic-light / overlay areas with padding per platform —
  `process.platform` reaches the renderer via the api bridge (`app.info`).

## 5. The CSS that separates "app" from "website in a box"

Chromium defaults are tuned for documents; desktop chrome needs overrides:

```css
/* Chrome (toolbars, sidebars, tabs) is furniture, not a document. */
body { user-select: none; cursor: default; }
/* Content and inputs opt back in. */
input, textarea, [contenteditable], .selectable { user-select: text; cursor: auto; }

/* Overlay-style scrollbars instead of 17px gutter bars. */
::-webkit-scrollbar { width: 10px; height: 10px; }
::-webkit-scrollbar-thumb { background: var(--border); border-radius: 5px; }
::-webkit-scrollbar-track { background: transparent; }

/* Focus rings for keyboard users only — a mouse click must not draw a ring. */
:focus { outline: none; }
:focus-visible { outline: 2px solid rgb(var(--primary)); outline-offset: 1px; }
```

The `user-select` rule is the big one: drag-selecting across a sidebar and
having button labels highlight blue is the instant "this is a webpage" giveaway.
Same family: `draggable="false"` on UI images (or `-webkit-user-drag: none`),
no pointer cursor on buttons if you want the native-toolbar feel (native
desktop buttons use the arrow — a judgment call, but make it once, globally),
and right-click should either show a real context menu (via main-process
`Menu.popup` over IPC, with roles for cut/copy/paste in text fields) or
nothing — never the browser default.

Keyboard: desktop users expect shortcuts for primary actions. Wire at minimum
Cmd/Ctrl+N (new), Cmd/Ctrl+, (settings) via menu accelerators — menu-declared
shortcuts show up in the menus themselves, which is how users discover them.

## 6. Perceived quality details

- **Zoom**: `{ role: "zoomIn" }/{ role: "zoomOut" }/{ role: "resetZoom" }` in
  the View menu — content zoom persists per-user expectations from every other
  desktop app; without the roles Ctrl+= does nothing.
- **Window state**: persist bounds on `close`, restore on create — validate
  the saved bounds intersect a current display
  (`screen.getDisplayMatching`), or a user who unplugged a monitor gets a
  window opening permanently off-screen.
- **min sizes** (`minWidth`/`minHeight`) that keep the layout intact; test the
  minimum — a layout that collapses at minimum size is #6 from
  `nextjs-setup-shadcn` in desktop form.

## Verify

1. Token grep from the web skills returns nothing; toggling dark flips every
   surface AND native context menus (`themeSource` wired).
2. Cold-start the packaged/dev app in dark mode: no white flash.
3. Devtools Network panel: zero requests to any external origin for
   fonts/CSS/JS.
4. On macOS (or ask someone with one): paste works in a text field — the menu
   exists. On Windows: Alt reveals the menu, titlebar overlay buttons work if
   frameless.
5. Drag-select from the middle of a toolbar: nothing highlights; drag the
   custom titlebar: window moves; click each titlebar button: it does its job
   and does not drag.
6. Tab through a screen: focus rings appear; click the same controls: none.
7. The built renderer CSS actually contains utilities: the asset is tens of
   KB and `grep -c "flex\|rounded" out/renderer/assets/*.css` is non-zero —
   ~1 KB of CSS means the Tailwind entry import/directives are missing (§1).

## Notes

- Multi-window apps repeat §3.2 and §4 per window — centralize window
  construction in one `createWindow(kind)` so nothing drifts.
- Related: `electron-create-app` (CSP this leans on), `electron-localization`
  (menu strings), `electron-state-management` (theme/bounds persistence),
  `NextJs/SetupShadCN` + `NextJs/ShadcnSelectTheme` (the web styling
  discipline that transfers wholesale).
