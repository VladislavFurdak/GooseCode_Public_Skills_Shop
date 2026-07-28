---
name: rn-modern-design
description: UI foundation for an Expo app — NativeWind (Tailwind for RN), design tokens with dark mode, loaded fonts, safe areas and native-feeling touch feedback. Use right after scaffolding, before building screens. Fixes the defaults that make RN apps look like prototypes — unstyled system fonts, hardcoded colors, content under the notch, dead-feeling buttons.
---

# Making an Expo app look designed, not defaulted

A fresh RN app renders in the platform system font, with pure-black-on-white
hardcoded colors, no dark mode, and content that slides under the status bar.
Each section below removes one of those tells. Do this before screens exist —
retrofitting tokens into written screens is the same repainting job the Next.js
skills warn about.

## 1. NativeWind — Tailwind classes in RN

NativeWind compiles Tailwind classes to RN styles at build time, so the styling
idiom matches the rest of this shop (Next.js + Tailwind + shadcn).

```bash
npx expo install nativewind react-native-reanimated react-native-safe-area-context
npm i -D tailwindcss@^3.4.0 prettier-plugin-tailwindcss
```

**Pin Tailwind to 3.4.x.** NativeWind 4 does not support Tailwind v4 — a plain
`npm i -D tailwindcss` pulls v4 and the build fails with config-resolution
errors that do not mention NativeWind at all. (When NativeWind ships a
Tailwind-v4-compatible major, check its own docs; until then this pin is
load-bearing.)

Four files must agree — miss one and classes silently do nothing:

```js
// tailwind.config.js
module.exports = {
  content: ["./app/**/*.{ts,tsx}", "./components/**/*.{ts,tsx}"],
  presets: [require("nativewind/preset")],
  theme: { extend: {} },
};
```

```css
/* global.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

```js
// babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [
      ["babel-preset-expo", { jsxImportSource: "nativewind" }],
      "nativewind/babel",
    ],
  };
};
```

```js
// metro.config.js
const { getDefaultConfig } = require("expo/metro-config");
const { withNativeWind } = require("nativewind/metro");
const config = getDefaultConfig(__dirname);
module.exports = withNativeWind(config, { input: "./global.css" });
```

Then `import "../global.css";` once, at the top of `app/_layout.tsx`.

After touching babel/metro config, restart with a cleared cache — stale
transforms are the classic "my className does nothing" cause:

```bash
npx expo start --clear
```

## 2. Tokens and dark mode

Define semantic colors once, as CSS variables, exactly like the shadcn skills —
never scatter `bg-white text-black` through screens (that is break #4 from
`nextjs-setup-shadcn`, and it kills dark mode here the same way):

```css
/* global.css, after the @tailwind directives */
:root {
  --background: 255 255 255;
  --foreground: 10 10 10;
  --card: 250 250 250;
  --muted-foreground: 115 115 115;
  --primary: 23 23 23;
  --primary-foreground: 250 250 250;
  --border: 229 229 229;
  --destructive: 220 38 38;
}
.dark:root {
  --background: 10 10 10;
  --foreground: 250 250 250;
  --card: 23 23 23;
  --muted-foreground: 163 163 163;
  --primary: 250 250 250;
  --primary-foreground: 23 23 23;
  --border: 38 38 38;
  --destructive: 248 113 113;
}
```

```js
// tailwind.config.js → theme.extend
colors: {
  background: "rgb(var(--background) / <alpha-value>)",
  foreground: "rgb(var(--foreground) / <alpha-value>)",
  card: "rgb(var(--card) / <alpha-value>)",
  "muted-foreground": "rgb(var(--muted-foreground) / <alpha-value>)",
  primary: { DEFAULT: "rgb(var(--primary) / <alpha-value>)", foreground: "rgb(var(--primary-foreground) / <alpha-value>)" },
  border: "rgb(var(--border) / <alpha-value>)",
  destructive: "rgb(var(--destructive) / <alpha-value>)",
},
```

Now screens say `bg-background text-foreground border-border`, and dark mode is
data, not a rewrite. NativeWind follows the OS setting by default; to let the
user override it, use `useColorScheme()` from `nativewind` and call
`setColorScheme("light" | "dark" | "system")` — persist the choice with the
settings store from `rn-state-management`.

Audit the same way the Next.js skill does — a healthy screen folder returns
nothing:

```bash
grep -rnE "(bg|text|border)-(white|black|gray|slate|zinc|neutral)-?[0-9]*" app components --include=*.tsx
```

Prebuilt components: `react-native-reusables` is the shadcn/ui port for
NativeWind (copy-into-repo components over accessible primitives). Its CLI and
registry move — check its docs for the current install command rather than
trusting a memorized one. For icons use `lucide-react-native` (same icon set as
the web side of this shop).

## 3. Fonts — the system font is the loudest "unstyled" tell

iOS renders SF, Android renders Roboto; the same screen looks like two
different apps. Load one brand font. The config-plugin route embeds fonts at
build time — no loading flash, no `useFonts` race:

```bash
npx expo install expo-font @expo-google-fonts/inter
```

```jsonc
// app.json → expo.plugins
[
  "expo-font",
  { "fonts": [
      "node_modules/@expo-google-fonts/inter/400Regular/Inter_400Regular.ttf",
      "node_modules/@expo-google-fonts/inter/500Medium/Inter_500Medium.ttf",
      "node_modules/@expo-google-fonts/inter/600SemiBold/Inter_600SemiBold.ttf"
  ]}
]
```

Wire it into Tailwind (`fontFamily: { sans: ["Inter_400Regular"] }`, plus
`font-medium`/`font-semibold` variants mapped to the 500/600 files) and put
`font-sans` on every `<Text>` via a shared `Text` component — RN has **no style
inheritance**: setting a font on a parent `View` styles nothing. A `components/
ui/text.tsx` that applies base classes is the standard fix; screens import it
instead of `react-native`'s Text.

Config-plugin fonts require a development build (they change the native app).
In Expo Go, fall back to `useFonts` from `expo-font` during early prototyping.

Weight discipline, same as the shadcn skill: headings 600 with tight tracking,
labels 500, body 400. On Android add `includeFontPadding: false` styling to the
shared Text component or vertical rhythm will look off by a few pixels.

## 4. Safe areas and the status bar

Content must never sit under the notch, the status bar, or the Android gesture
bar — and on current SDKs Android is edge-to-edge by default, so this is no
longer an iPhone-only concern.

- Wrap the app in `SafeAreaProvider` (root layout — the scaffold's template
  already does), then consume `useSafeAreaInsets()` in screen containers:
  `style={{ paddingTop: insets.top, paddingBottom: insets.bottom }}`.
- Do **not** import `SafeAreaView` from `react-native` — it is iOS-only and
  deprecated. Use `react-native-safe-area-context` exclusively.
- Never hardcode `paddingTop: 44` — that number is wrong on half the devices.
- Style the status bar with `expo-status-bar`'s `<StatusBar style="auto" />` so
  icons flip with the theme.

## 5. Touch feedback — dead buttons read as broken

Use `Pressable`, never the legacy `TouchableOpacity`/`TouchableHighlight`, and
give every tappable thing feedback and a real hit target:

```tsx
<Pressable
  className="h-12 items-center justify-center rounded-xl bg-primary active:opacity-80"
  android_ripple={{ color: "rgba(255,255,255,0.2)" }}
  hitSlop={8}
>
  <Text className="font-semibold text-primary-foreground">Save</Text>
</Pressable>
```

- `active:` opacity for iOS-feel, `android_ripple` for Android-feel — one
  component, both platforms feel native (details in `rn-cross-platform`).
- Minimum touch target 44×44pt (`h-12` ≈ 48). `hitSlop` extends small icons.
- Long lists: `FlashList` (`@shopify/flash-list`) instead of `FlatList` once a
  list can exceed a couple of screens — same API, no blank-cell flicker.
- Animations: `react-native-reanimated` is already installed by the scaffold —
  use it (or its `Animated.View` entering/exiting presets) rather than the
  core `Animated` API.

## Verify

`tsc --noEmit` passes, then look at screens — every check here is visual:

1. A `text-primary` element actually changes color when the OS theme flips —
   proves tokens + dark mode wiring end-to-end.
2. Type renders in Inter on BOTH platforms (Android falling back to Roboto
   means the font file name in Tailwind config doesn't match the loaded name).
3. Nothing sits under the status bar / notch / gesture bar on either platform.
4. Every button visibly reacts to touch.
5. The grep in §2 returns nothing outside `components/ui`.

## Notes

- One `--clear` restart after any config-file change; Metro caches transforms
  aggressively and stale caches mimic every bug above.
- Related: `expo-create-app` (run first), `rn-cross-platform` (platform-feel
  details), `rn-localization` (the shared Text component is also where
  translated strings flow through).
