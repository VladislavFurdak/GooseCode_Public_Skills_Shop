---
name: rn-cross-platform
description: The iOS/Android differences that actually bite in an Expo app — shadows, keyboard, safe areas and edge-to-edge, the Android back button, fonts and text metrics, touch feedback — and the mechanisms for platform-specific code. Use while building screens, and as a checklist before calling any screen done. "Works on the emulator" means one platform; this skill is the other one.
---

# One codebase, two platforms that disagree

React Native shares logic and layout, not platform behavior. The failures
below are all of the same species: **correct on the platform you develop on,
broken on the one you don't look at.** On Windows that unwatched platform is
almost always iOS (`expo-create-app` §4) — which is exactly why each item here
names its symptom on both.

## 1. Mechanisms for platform-specific code (prefer them in this order)

**1. Don't branch at all.** Most "iOS vs Android" code is a missing
abstraction — the libraries in these skills (safe-area-context, expo-*,
react-navigation) already normalize the platforms. Reach for a branch last.

**2. Inline, for one prop:**

```ts
import { Platform } from "react-native";
behavior={Platform.OS === "ios" ? "padding" : undefined}
...Platform.select({ ios: { shadowOpacity: 0.15 }, android: { elevation: 4 } })
```

**3. File split, when a component diverges structurally:**

```
components/DatePicker.ios.tsx
components/DatePicker.android.tsx
components/DatePicker.tsx        ← optional web/fallback
```

`import { DatePicker } from "@/components/DatePicker"` — Metro resolves the
platform file automatically; the import site never knows. Keep the exported
prop types identical across the files or the split leaks into every caller.

## 2. Shadows — the most visible divergence

iOS `shadow*` props do nothing on Android; Android `elevation` does nothing on
iOS. Half-styled cards — flat on one platform — are the result.

The modern answer: RN 0.76+ (New Architecture) supports the CSS-style
`boxShadow` style prop **cross-platform** — NativeWind `shadow-*` classes
compile appropriately as well. Use one of those. If a library forces the old
props, always ship the pair:

```ts
...Platform.select({
  ios: { shadowColor: "#000", shadowOffset: { width: 0, height: 2 }, shadowOpacity: 0.1, shadowRadius: 8 },
  android: { elevation: 3 },
})
```

Android elevation shadows are tinted by the view's `backgroundColor` — a
"transparent" card with elevation renders no shadow at all on Android. Give
elevated views an opaque background.

## 3. Keyboard — inputs hidden behind it

The keyboard covers the bottom half of the screen and the platforms handle it
oppositely: Android resizes the window (mostly fine), iOS overlays (your
bottom input is now invisible).

```tsx
<KeyboardAvoidingView
  behavior={Platform.OS === "ios" ? "padding" : undefined}
  keyboardVerticalOffset={headerHeight}   // the height of any header above — measure, don't guess
  className="flex-1"
>
```

`behavior="height"` on iOS and `"padding"` on Android are the wrong-way-around
combinations that jitter. For chat-style screens (input pinned to bottom,
scrolling list above), `KeyboardAvoidingView` runs out — use
`react-native-keyboard-controller`'s `KeyboardAwareScrollView`/`KeyboardStickyView`
(native module → dev build, `expo-create-app` §3).

Two habits regardless: `keyboardShouldPersistTaps="handled"` on any ScrollView
containing inputs (otherwise the first tap on a button only dismisses the
keyboard and the user must tap twice), and dismiss on scroll for feeds
(`keyboardDismissMode="on-drag"`).

## 4. Edge-to-edge and safe areas — Android joined the notch club

`rn-modern-design` §4 sets up the insets machinery; the cross-platform point:
current Expo SDKs render Android **edge-to-edge by default** — content draws
under the status bar and the gesture-nav bar, exactly like iOS notches. Code
that only pads for iOS (`Platform.OS === "ios" ? insets.top : 0` — a pattern
copied from old tutorials) is now wrong. Use the insets unconditionally; they
are correct on every device of both platforms, including ones with no cutout
at all.

Bottom insets matter as much as top: a floating action button or tab bar at
`bottom: 0` sits behind the Android gesture bar / iPhone home indicator.
`paddingBottom: insets.bottom` on the container fixes both at once.

## 5. Android back — the button iOS doesn't have

Hardware/gesture back pops the expo-router stack correctly with zero code —
**test it anyway on every flow**: the failure is UX-level, e.g. back from the
first post-login screen returning to the login form (fix: `router.replace`,
not `router.push`, across auth boundaries — a bug invisible on iOS, glaring
on Android).

Intercept back only for data-loss moments (unsaved form), via React
Navigation's `usePreventRemove`, which also catches iOS swipe-back and header
back — one interception, all three gestures. Reach for raw `BackHandler` only
outside navigation contexts (e.g. closing a custom overlay), and remember iOS
users get no equivalent — the overlay still needs a visible close button.

## 6. Text and fonts — same string, different pixels

- Without loaded fonts, iOS renders SF and Android renders Roboto — different
  widths, different line breaks, layouts that "randomly" differ. The brand
  font from `rn-modern-design` §3 is also the cross-platform fix.
- Android adds invisible vertical padding to text (`includeFontPadding`) —
  vertically-centered labels sit ~2px low. The shared Text component sets
  `includeFontPadding: false` plus an explicit `lineHeight` once, globally.
- `fontWeight: "600"` with a loaded custom font silently falls back on
  Android if only the 400 file is loaded — Android needs the actual 500/600
  font files registered (the config-plugin list in `rn-modern-design` §3),
  it does not synthesize weights the way iOS does.
- `allowFontScaling` is on by default (good — accessibility). Test at 1.3×
  system font size; fixed-height containers that clip scaled text are the
  common casualty. Prefer `minHeight` over `height` for text containers.

## 7. Touch, gestures, haptics

- `Pressable` with both `android_ripple` and `active:opacity` (the
  `rn-modern-design` §5 recipe) is the whole answer to "feels native on
  both" — ripple on Android, dim on iOS, one component.
- Ripple bleeds outside rounded corners unless the pressable (or a wrapper)
  has `overflow-hidden` with the matching radius.
- iOS edge-swipe-back and Android gesture-nav both own screen-edge gestures —
  custom horizontal swipes (carousels, swipe-to-delete) near edges need
  `activeOffsetX` tuning in gesture-handler, or they fight the system. Keep
  primary actions off swipe-only affordances; Android users especially expect
  a visible alternative.
- `expo-haptics` on button-presses feels premium on iOS and is inconsistent
  across Android hardware — call it unconditionally (it degrades safely) but
  never make feedback *only* haptic.

## 8. Web as a bonus target — cheap if you stay honest

`npx expo start` → `w` runs the same app via react-native-web. It stays cheap
under two conditions: only RN primitives (`View`/`Text`/`Pressable` — never
`div`), and guard the modules with no web implementation
(haptics, secure-store, mmkv) behind `Platform.OS === "web"` checks or `.web.ts`
file splits. Expo Router serves each route as a URL, which makes web builds a
genuinely useful preview/demo channel even when web isn't a shipping target —
but state it plainly: if web IS a shipping target, that decision belongs in
the plan, not as a side effect.

## Verify — the two-platform pass

Before calling a screen done, run it on Android **and** iOS (Windows matrix:
emulator + physical iPhone). Walk one list:

1. Cards/elevated surfaces show shadows on both.
2. Focus every input: keyboard covers nothing; buttons tappable in one tap
   while the keyboard is up.
3. Nothing under status bar / notch / gesture bar, top and bottom.
4. Android back from every screen lands somewhere sensible; auth screens
   don't resurface.
5. Text: brand font on both, weights render, nothing clips at 1.3× font scale.
6. Buttons give visible touch feedback on both.

Ten minutes per screen; every item above is invisible in code review and
obvious on a device.

## Notes

- Platform differences belong at the edges (shared components like Text,
  Button, Screen) — if `Platform.select` appears inside feature screens,
  a shared component is missing.
- Related: `rn-modern-design` (the shared components this skill keeps
  pointing at), `expo-create-app` §4 (the Windows device matrix),
  `rn-build-ios-android` (release-mode testing on both platforms).
