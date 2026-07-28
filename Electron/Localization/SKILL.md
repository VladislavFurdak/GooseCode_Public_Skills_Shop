---
name: electron-localization
description: Localization for an Electron app with typed message dictionaries — en.ts is the compile-time contract every locale must satisfy, plus locale resolution with regional fallbacks, a React provider, and the part web guides miss - the application menu lives in the MAIN process and must be rebuilt on language change. Use when the app needs more than one language. Pattern verified in production across 12 locales (GooseCode).
---

# Desktop localization the compiler enforces

Same architecture as `rn-localization`, because the same reasoning holds
doubled: an Electron app bundles everything, runs offline, and has no
bundle-size pressure that would justify async catalog loading — i18next-style
runtime machinery solves problems this platform does not have. Typed
dictionaries compile in, cost zero at runtime, and turn "missing translation"
into a build failure. The reference app ships 12 locales this way; its en.ts
header states the conventions this skill encodes.

## 1. Canonical dictionary and the type contract

```ts
// src/shared/i18n/en.ts
/**
 * English — the canonical dictionary. Its shape is the `Messages` contract
 * every other locale is type-checked against: adding a key here forces a
 * translation in every locale file.
 * Conventions: `{name}` placeholders are filled by fmt(); brand and technical
 * tokens (product name, "Supabase", model ids…) stay untranslated everywhere.
 */
export const en = {
  menu: { file: "File", edit: "Edit", view: "View", settings: "Settings…", quit: "Quit" },
  sidebar: { newItem: "New item", empty: "Nothing here yet.", delete: "Delete" },
  update: {
    available: "Version {version} is available",
    ready: "Update {version} is ready to install",
    later: "Later",
  },
  counts: {
    files: (n: number) => (n === 1 ? "1 file" : `${n} files`),
  },
};
export type Messages = typeof en;
```

```ts
// src/shared/i18n/uk.ts
import type { Messages } from "./en";
export const uk: Messages = { /* every key, or it does not compile */ };
```

The annotation `: Messages` is the whole mechanism — no `Partial`, no
`as`-casts, or missing keys silently fall back to nothing at all (there IS no
runtime fallback layer here; the compiler is the fallback layer). Plurals are
plain functions per locale (see `rn-localization` §1 for a full Slavic
example) — real code beats ICU strings that nothing type-checks.

**Location matters: `src/shared/`, not the renderer.** §4 is why.

## 2. Registry, resolution, formatting

```ts
// src/shared/i18n/index.ts
export type LocaleId = "en" | "uk" | "de";

export const LOCALES: { id: LocaleId; label: string; messages: Messages }[] = [
  { id: "en", label: "English (United States)", messages: en },
  { id: "uk", label: "Українська (Україна)", messages: uk },   // native-name labels:
  { id: "de", label: "Deutsch (Deutschland)", messages: de },  // the picker must be
];                                                             // readable in the WRONG language

export function resolveLocale(tag: string | undefined): LocaleId {
  if (!tag) return "en";
  const lower = tag.toLowerCase();
  if ((LOCALES as { id: string }[]).some((l) => l.id === lower)) return lower as LocaleId;
  // Regional fallbacks are product decisions — make them explicit, per family:
  // e.g. "pt" → "pt-BR"; "es" → "es-ES", other es-* → "es-419" (production example).
  const base = lower.split("-")[0];
  return (LOCALES.find((l) => l.id === base)?.id) ?? "en";
}

export function fmt(template: string, params: Record<string, string | number>): string {
  return template.replace(/\{(\w+)\}/g, (_, k: string) => (k in params ? String(params[k]) : `{${k}}`));
}
```

System locale: `navigator.language` in the renderer and `app.getLocale()` in
main report the same user preference — use whichever side needs it, through
the same `resolveLocale`.

Dates and numbers go through `Intl` **with the app locale passed explicitly**
(`new Intl.DateTimeFormat(locale, …)`), centralized in helpers next to the
dictionaries. A bare `toLocaleString()` follows the OS locale, and an app set
to English on a German system starts mixing "1.024,5" into English screens.

## 3. Renderer: provider, hook, persisted override

Standard context (`rn-localization` §3 has the full listing — same code
minus the Expo import): `I18nProvider` holds `LocaleId | "system"`, resolves
`"system"` through `resolveLocale(navigator.language)`, exposes
`{ t, locale, setLocale }`. Two Electron specifics:

- The override persists in the **main-process settings store**
  (`electron-state-management`), not localStorage — set via
  `window.api.settings.set({ locale })`, hydrated at boot, updated by the
  `settings:changed` push. Locale is app state like the theme, and it must be
  readable by main (§4) — localStorage isn't.
- Keep `"system"` stored as the distinct value so the app tracks OS-language
  changes until the user explicitly chooses.

Usage everywhere: `t.sidebar.newItem`,
`fmt(t.update.available, { version })`, `t.counts.files(n)`.

## 4. The part web guides miss: main process UI speaks too

The application menu, tray tooltips, native dialog buttons
(`dialog.showMessageBox`), and the macOS dock menu are built in **main** —
the renderer's React context never touches them. This is why dictionaries
live in `src/shared/`: main imports the same `LOCALES`/`fmt` directly.

Menus do not re-render; rebuild on change:

```ts
// src/main/menu.ts
export function applyMenu(locale: LocaleId): void {
  const t = LOCALES.find((l) => l.id === locale)!.messages;
  Menu.setApplicationMenu(Menu.buildFromTemplate([
    // role-based items (editMenu etc. — electron-modern-design §4) label themselves;
    // your custom items take t.menu.* labels:
    { label: t.menu.file, submenu: [{ label: t.menu.settings, accelerator: "CmdOrCtrl+,", click: openSettings }] },
    { role: "editMenu" },
  ]));
}
```

Call `applyMenu` at startup and again inside the settings-store's change
handler when `locale` changes. A language switch that translates the window
but leaves the menu bar in English is the exact half-translated look this
section exists to prevent. Same trigger rebuilds the tray menu if one exists.

Dialog example: `dialog.showMessageBox({ message: t.update.ready
&& fmt(...), buttons: [t.update.later, ...] })` — never hardcode dialog
buttons; they are the strings users screenshot.

## 5. Keeping literals out

```bash
grep -rnE ">[A-Za-z][^<>{}]*<" src/renderer --include=*.tsx | grep -vE "(className|import|//)" | head -50
```

Hits inside JSX are hardcoded strings; healthy output is brand tokens and
punctuation only. Run it per feature, not once at the end. In main, the
equivalent offenders are `label:` and `message:`/`buttons:` literals —
grep `src/main` for those keys.

## Verify

1. `tsc --noEmit` — the completeness proof for all locales at once.
2. Switch language in-app: window strings, **the menu bar**, and a native
   dialog all flip without restart.
3. Plural spot-check with counts 1 / 3 / 7 in a Slavic locale, if shipped.
4. Set override to "system", relaunch with a different OS display language
   (or mock `navigator.language`): the app follows; explicit choice survives
   relaunch.
5. A date and a large number render per the app locale, not the OS one.

## Notes

- Translation workflow: new locale = copy `en.ts`, translate values, compiler
  reviews shape. This is also the safe shape for LLM translation of the
  file — structural drift cannot survive `tsc`. Keep per-file header comments
  (what stays untranslated) so future translators — human or model — inherit
  the conventions.
- RTL on desktop is CSS (`dir="rtl"` on `<html>`, logical properties) — no
  restart dance like RN. Set `document.documentElement.dir` alongside the
  locale if an RTL language ships.
- Related: `rn-localization` (same pattern, mobile specifics),
  `electron-state-management` (where the override lives),
  `electron-modern-design` §4 (the menu this skill translates).
