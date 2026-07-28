---
name: rn-localization
description: Localization for an Expo app with typed message dictionaries — the English dictionary is the type contract every locale compiles against, so a missing translation is a compile error, not a runtime fallback. Covers system-locale detection, user override, plurals without ICU, Intl dates/numbers, and RTL. Use when the app needs more than one language, ideally before many screens exist.
---

# Localization that the type-checker enforces

No i18next, no ICU message syntax, no async catalog loading. A mobile app ships
its strings inside the bundle, so the web problems those libraries solve do not
exist here. What you want instead is the guarantee that **every locale has
every key** — and TypeScript gives you that for free if the dictionary is
typed. This is the same pattern as `electron-localization`; the two skills
share a philosophy on purpose, and it has survived production with 12 locales.

## 1. The canonical dictionary

English is the source of truth. Its **shape** is the contract:

```ts
// i18n/en.ts
export const en = {
  common: {
    save: "Save",
    cancel: "Cancel",
    retry: "Retry",
  },
  home: {
    title: "Home",
    greeting: "Hello, {name}",
    // Plurals are plain functions — no ICU syntax to learn or mis-parse.
    itemCount: (n: number) => (n === 1 ? "1 item" : `${n} items`),
  },
  settings: {
    title: "Settings",
    language: "Language",
  },
};

export type Messages = typeof en;
```

Every other locale imports the type. **This line is the entire mechanism:**

```ts
// i18n/uk.ts
import type { Messages } from "./en";

export const uk: Messages = {
  common: { save: "Зберегти", cancel: "Скасувати", retry: "Повторити" },
  home: {
    title: "Головна",
    greeting: "Привіт, {name}",
    itemCount: (n: number) => {
      const mod10 = n % 10, mod100 = n % 100;
      if (mod10 === 1 && mod100 !== 11) return `${n} елемент`;
      if (mod10 >= 2 && mod10 <= 4 && (mod100 < 10 || mod100 >= 20)) return `${n} елементи`;
      return `${n} елементів`;
    },
  },
  settings: { title: "Налаштування", language: "Мова" },
};
```

Add a key to `en.ts` and **every locale file stops compiling until translated**.
Delete a key and stale translations surface as errors. That is the feature; do
not weaken it with `Partial<Messages>` or `as Messages` — either one turns
missing translations back into silent runtime English.

Slavic plural rules fit in five lines of real code per message, where ICU would
be an opaque string nobody can type-check. Keep brand names, product terms and
technical tokens untranslated in every locale.

Placeholders use `{name}` filled by one tiny helper:

```ts
// i18n/index.ts
export function fmt(template: string, params: Record<string, string | number>): string {
  return template.replace(/\{(\w+)\}/g, (_, k: string) =>
    k in params ? String(params[k]) : `{${k}}`,
  );
}
```

## 2. Registry and locale resolution

```ts
// i18n/index.ts (continued)
import { en, type Messages } from "./en";
import { uk } from "./uk";

export type LocaleId = "en" | "uk";

export const LOCALES: { id: LocaleId; label: string; messages: Messages }[] = [
  { id: "en", label: "English" },
  { id: "uk", label: "Українська" },   // labels in the language's OWN name — the
].map((l) => ({ ...l, messages: l.id === "en" ? en : uk }));   // picker must be readable
                                                               // to someone lost in the wrong language
export function resolveLocale(tag: string | undefined): LocaleId {
  if (!tag) return "en";
  const base = tag.toLowerCase().split("-")[0];
  return (LOCALES.find((l) => l.id === base)?.id as LocaleId) ?? "en";
}
```

System locale comes from `expo-localization`:

```bash
npx expo install expo-localization
```

```ts
import { getLocales } from "expo-localization";
const systemTag = getLocales()[0]?.languageTag;   // e.g. "uk-UA"
```

`getLocales()` is ordered by user preference — take `[0]`. Map regional tags
deliberately when you support several variants of one language (`pt` → `pt-BR`,
`es` → `es-419` vs `es-ES`); a bare `startsWith` that picks the wrong regional
variant reads as sloppy to native speakers.

## 3. Context, hook, override

```tsx
// i18n/context.tsx
const I18nContext = createContext<{ t: Messages; locale: LocaleId; setLocale: (l: LocaleId | "system") => void } | null>(null);

export function I18nProvider({ children }: { children: React.ReactNode }) {
  const [override, setOverride] = useState<LocaleId | "system">(loadSavedLocale() ?? "system");
  const locale = override === "system" ? resolveLocale(getLocales()[0]?.languageTag) : override;
  const setLocale = (l: LocaleId | "system") => { setOverride(l); saveLocale(l); };
  const t = LOCALES.find((x) => x.id === locale)!.messages;
  return <I18nContext.Provider value={{ t, locale, setLocale }}>{children}</I18nContext.Provider>;
}

export function useI18n() {
  const ctx = useContext(I18nContext);
  if (!ctx) throw new Error("useI18n outside I18nProvider");
  return ctx;
}
```

Persist the override with the storage from `rn-state-management`; keep
`"system"` as a distinct saved value so the app keeps following the OS until
the user explicitly picks a language. Usage:

```tsx
const { t } = useI18n();
<Text>{fmt(t.home.greeting, { name: user.name })}</Text>
<Text>{t.home.itemCount(items.length)}</Text>
```

## 4. Dates and numbers — Intl, with the locale passed explicitly

Hermes ships `Intl` on both platforms on current RN. Always pass the app
locale; a bare `toLocaleDateString()` follows the *device*, so a user who set
your app to English on a Ukrainian phone gets mixed-language screens:

```ts
new Intl.DateTimeFormat(locale, { dateStyle: "medium" }).format(date);
new Intl.NumberFormat(locale, { style: "currency", currency: "USD" }).format(n);
```

Centralize these as `formatDate`/`formatMoney` helpers that read the context
locale — scattered ad-hoc `Intl` calls are how one screen ends up formatted
differently from the rest.

## 5. RTL — decide early

Arabic/Hebrew support is cheap if styles are logical from day one and
expensive to retrofit:

- Use logical properties everywhere: `ps-4`/`pe-4` (NativeWind), `text-start`,
  `flex-row` (which RN mirrors automatically) — never `pl-4`/`text-left` for
  directional layout.
- Enable native RTL support in `app.json`: `"expo": { "extra": { "supportsRTL": true } }`
  (read by `expo-localization`'s config plugin — requires a rebuild).
- Runtime state: `I18nManager.isRTL`. Switching direction at runtime requires
  `I18nManager.forceRTL(...)` **plus an app reload** — direction is applied at
  startup, not reactively. Ship direction-switching as "takes effect on
  restart" and it stops being a bug report.

If the locale list will plausibly never include an RTL language, skip the
forceRTL machinery but keep the logical-property habit — it costs nothing.

## 6. Keep literals out of screens

The pattern only works if screens actually use it. Audit for raw JSX literals:

```bash
grep -rnE "<Text[^>]*>[^{<]" app components --include=*.tsx | grep -v "components/ui"
```

Hits are hardcoded strings. A healthy codebase returns only genuinely
untranslatable tokens (brand names, "—", units). Run it before calling any
screen done — retrofitting keys after review is double work.

## Verify

1. `tsc --noEmit` — this IS the completeness test for every locale.
2. Switch the language in-app: every visible string flips, including plurals
   with counts 1 / 3 / 7 (that spread catches Slavic rules).
3. Set the override back to "system", change the OS language — the app follows.
4. Dates/numbers render in the app locale, not the device locale.

## Notes

- Translating a new locale is mechanical: copy `en.ts`, change the values,
  keep the shape — the compiler reviews the result. This is also the ideal
  shape for LLM-assisted translation, since drift is caught by types.
- If the app later genuinely needs ICU/TMS workflows (external translators,
  dozens of locales), migrate to `i18n-js` + `expo-localization` — the typed
  dictionary converts mechanically. Do not start there.
- Related: `electron-localization` (same pattern, desktop), `rn-modern-design`
  (the shared Text component all strings flow through), `rn-state-management`
  (persisting the override).
