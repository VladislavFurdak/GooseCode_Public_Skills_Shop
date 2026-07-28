---
name: rn-state-management
description: State architecture for an Expo app — TanStack Query for server data, Zustand for client state, MMKV (or AsyncStorage in Expo Go) for persistence. Use before the first screen that holds data. Prevents the standard failure — server data copied into a global store, going stale, refetched by hand, and re-rendering everything on every keystroke.
---

# Two kinds of state, two tools — never one big store

Almost everything on a mobile screen is a **cache of the server** (lists,
profiles, messages) or **local UI/session state** (theme, drafts, auth token,
toggles). The standing mistake is one Redux-style store holding both, with
hand-written fetch/loading/error plumbing around it. Split instead:

| Kind | Tool | Persisted? |
|---|---|---|
| Server data (anything fetched) | TanStack Query | no — refetched |
| Client state (settings, session, UI) | Zustand | selectively |
| Form input | react-hook-form (or local `useState`) | no |

Redux Toolkit is not wrong, it is simply more machinery than a small app needs;
nothing below blocks introducing it later. What IS wrong is putting fetched
data into Zustand — you re-implement caching, invalidation, deduplication and
staleness by hand, badly, one bug at a time.

## 1. Server state — TanStack Query

```bash
npm i @tanstack/react-query
```

Provider at the root, once:

```tsx
// app/_layout.tsx
const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 60_000, retry: 2 } },
});
// wrap the layout in <QueryClientProvider client={queryClient}>
```

Queries own fetching, caching, retries and loading/error states; screens
declare what they need:

```tsx
const { data, isPending, error, refetch } = useQuery({
  queryKey: ["todos", userId],
  queryFn: () => fetchTodos(userId),
});
```

Mutations invalidate instead of hand-patching sibling caches:

```tsx
const qc = useQueryClient();
const addTodo = useMutation({
  mutationFn: createTodo,
  onSuccess: () => qc.invalidateQueries({ queryKey: ["todos"] }),
});
```

**Two RN-specific wirings web tutorials omit** — without them "refetch on
focus/reconnect" silently never fires, because RN has no `window` focus or
browser online events:

```ts
// app focus → query focus
import { AppState } from "react-native";
import { focusManager, onlineManager } from "@tanstack/react-query";

AppState.addEventListener("change", (s) => focusManager.setFocused(s === "active"));

// connectivity → query online state
import * as Network from "expo-network";        // npx expo install expo-network
onlineManager.setEventListener((setOnline) => {
  const sub = Network.addNetworkStateListener((s) => setOnline(!!s.isConnected));
  return () => sub.remove();
});
```

Pull-to-refresh maps directly: `<FlatList refreshing={isRefetching}
onRefresh={refetch} …>`.

Key conventions: keys are arrays from general to specific
(`["todos"]`, `["todos", userId]`, `["todos", userId, todoId]`) so one
`invalidateQueries({ queryKey: ["todos"] })` sweeps every dependent view.

## 2. Client state — Zustand

```bash
npm i zustand
```

```ts
// stores/settings.ts
import { create } from "zustand";
import { persist, createJSONStorage } from "zustand/middleware";
import { storage } from "./storage";   // §3

interface SettingsState {
  theme: "light" | "dark" | "system";
  locale: string;                       // "system" or a LocaleId — see rn-localization
  setTheme: (t: SettingsState["theme"]) => void;
  setLocale: (l: string) => void;
}

export const useSettings = create<SettingsState>()(
  persist(
    (set) => ({
      theme: "system",
      locale: "system",
      setTheme: (theme) => set({ theme }),
      setLocale: (locale) => set({ locale }),
    }),
    { name: "settings", storage: createJSONStorage(() => storage) },
  ),
);
```

Consume with **selectors**, never the whole store — `useSettings()` bare
subscribes the component to every field and produces the re-render storms
Zustand exists to avoid:

```ts
const theme = useSettings((s) => s.theme);            // re-renders only on theme
const { a, b } = useSettings(useShallow((s) => ({ a: s.a, b: s.b })));  // several fields
```

Small focused stores (`useSettings`, `useSession`, `useComposerDraft`) beat one
app-wide store: independent subscriptions, no reducer ceremony, and each store
decides for itself whether it persists.

## 3. Persistence — MMKV, with an Expo Go escape hatch

`react-native-mmkv` is synchronous and ~30× faster than AsyncStorage — but it
is a **native module, so it crashes in Expo Go** (`expo-create-app` §3). Both
adapters below satisfy zustand's `StateStorage`; pick by what you run on:

```ts
// stores/storage.ts — development build (the end state)
import { MMKV } from "react-native-mmkv";
const mmkv = new MMKV();
export const storage = {
  getItem: (k: string) => mmkv.getString(k) ?? null,
  setItem: (k: string, v: string) => mmkv.set(k, v),
  removeItem: (k: string) => mmkv.delete(k),
};
```

```ts
// stores/storage.ts — while still in Expo Go
import AsyncStorage from "@react-native-async-storage/async-storage";
export const storage = AsyncStorage;   // async — see the hydration note below
```

The swap is one file; stores don't change. Current MMKV majors require the New
Architecture — fine, it is the Expo default.

**Async-storage hydration is not instant.** With AsyncStorage (and with MMKV it
is instant, but don't rely on that in shared code), a persisted store renders
its defaults for the first frames, then rehydrates. Any boot decision that
reads persisted state — initial route, theme, "seen onboarding?" — must wait:

```ts
const hydrated = useSettings.persist.hasHydrated();
if (!hydrated) return <SplashPlaceholder />;   // or listen via onFinishHydration
```

Skipping this is the classic "app flashes light theme / shows onboarding again
for a frame" bug.

Persist **only what must survive a restart** (settings, session, drafts).
Never persist server data via Zustand — if offline-first caching is truly
needed, that is TanStack Query's persister plugin, a deliberate later step.
Secrets (tokens) belong in `expo-secure-store`, not MMKV/AsyncStorage —
`rn-supabase-setup` handles the auth session's storage specifically.

## 4. Forms

Controlled `useState` per field is fine for one or two inputs. Beyond that:

```bash
npm i react-hook-form
```

RHF keeps keystrokes out of any global store — typing re-renders one input,
not a screen. Wire RN inputs through its `Controller`. On submit, hand the
values to a TanStack mutation. A form is neither server state nor app state;
it should die with the screen (drafts being the deliberate exception — a tiny
persisted Zustand store).

## Verify

1. `tsc --noEmit` passes.
2. Kill and relaunch the app: settings survive, and there is **no
   wrong-value flash** on boot (hydration gate works).
3. Airplane mode on → queries pause, no error spam; off → data refetches
   without user action (proves the onlineManager wiring).
4. Background the app, change server data elsewhere, foreground it → the list
   updates (proves the focusManager wiring).
5. Type into a form while React DevTools highlights re-renders: only the input
   re-renders.

## Notes

- Keep stores in `stores/`, one file per store; queries live beside their
  feature or in `queries/`. No `store/index.ts` barrel re-exporting
  everything — it recreates the monolith import graph.
- Related: `rn-supabase-setup` (its queries plug into TanStack, its session
  into the pattern here), `rn-localization` (locale override persists via
  `useSettings`), `expo-create-app` §3 (why MMKV forces a dev build).
