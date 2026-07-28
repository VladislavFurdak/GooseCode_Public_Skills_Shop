---
name: rn-supabase-setup
description: Supabase in an Expo app — client with the right auth storage, session refresh tied to AppState, email + OAuth deep-link flows, route protection with expo-router, and the local-stack networking that mobile breaks (127.0.0.1 is the phone, not your machine). Use before writing any data or auth code. Schema/migrations/RLS are shared with the Next.js supabase-setup skill — do not duplicate them here.
---

# Supabase on a phone

The database side — local stack, migrations, RLS, generated types — is
identical to the web and already written down: follow `NextJs/SupabaseSetup`
§2, §5, §6 for that. This skill covers what mobile changes: **the client
config, the network path to your dev machine, and auth without redirects**.

One rule outranks everything: **the app binary is public.** Anyone can unzip
an APK. Only the URL and the anon key may exist in the app; RLS is the entire
security model. There is no "server side" in this app to hide a
`service_role` key in — privileged operations belong in Edge Functions or your
own backend, never client code.

## 1. Install and client

```bash
npx expo install @supabase/supabase-js @react-native-async-storage/async-storage react-native-url-polyfill
```

```ts
// lib/supabase.ts
import "react-native-url-polyfill/auto";        // MUST be imported before createClient
import AsyncStorage from "@react-native-async-storage/async-storage";
import { createClient } from "@supabase/supabase-js";
import { AppState } from "react-native";
import type { Database } from "@/types/database";

export const supabase = createClient<Database>(
  process.env.EXPO_PUBLIC_SUPABASE_URL!,
  process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY!,
  {
    auth: {
      storage: AsyncStorage,
      autoRefreshToken: true,
      persistSession: true,
      detectSessionInUrl: false,   // there is no URL to detect a session in
      flowType: "pkce",            // required for the OAuth/deep-link flow in §4
    },
  },
);

// Tokens refresh only while the app is foregrounded — this wiring is what
// keeps a user logged in across days of backgrounding.
AppState.addEventListener("change", (state) => {
  if (state === "active") supabase.auth.startAutoRefresh();
  else supabase.auth.stopAutoRefresh();
});
```

Every piece of that config exists for a reason: RN has no `window.location`
(→ `detectSessionInUrl: false`), no localStorage (→ explicit `storage`), and
suspends timers in background (→ the AppState hook). The URL polyfill patches
gaps in RN's URL implementation that supabase-js trips over — a missing import
surfaces as unrelated-looking runtime errors deep inside the client.

Env vars go in `.env` with the `EXPO_PUBLIC_` prefix (inlined at build time,
same visibility rules as `NEXT_PUBLIC_`):

```bash
# .env
EXPO_PUBLIC_SUPABASE_URL=http://192.168.1.20:54321   # see §2 for why not 127.0.0.1
EXPO_PUBLIC_SUPABASE_ANON_KEY=<from `npx supabase status`>
```

Restart `expo start` after changing `.env` — values are baked in, not read live.

## 2. Reaching the local stack — the trap that wastes the most time

`npx supabase start` serves on `127.0.0.1:54321` **of your computer**. On a
device, `127.0.0.1` is the device itself. Symptom: `Network request failed`
with a stack that looks like an auth bug; nothing is wrong with the code.

| Where the app runs | URL that reaches your machine |
|---|---|
| iOS simulator (macOS) | `http://127.0.0.1:54321` — shares the host loopback |
| Android emulator | `http://10.0.2.2:54321` — the emulator's alias for the host |
| Physical device (the Windows-realistic path) | `http://<LAN IP>:54321`, e.g. `192.168.1.20` — `ipconfig`, same Wi-Fi, firewall allowing the port |

Practical default: use the LAN IP everywhere — it works for all three. The
cost is that `.env` differs per developer machine, which is why `.env` is
gitignored. If the LAN path is blocked (guest Wi-Fi isolation, VPN), stop
fighting it and point the app at a hosted Supabase project instead — creating
one is the user's job, exactly as `NextJs/SupabaseSetup` §7 words it.

Plain `http://` works in dev on both platforms under Expo's defaults; the
hosted project is `https://` so release builds never depend on cleartext
exceptions.

## 3. Email/password auth and route protection

`signUp` / `signInWithPassword` / `signOut` work unchanged. Wire the session
to navigation once, in the root layout:

```tsx
// app/_layout.tsx
const [session, setSession] = useState<Session | null>(null);
const [ready, setReady] = useState(false);

useEffect(() => {
  supabase.auth.getSession().then(({ data }) => { setSession(data.session); setReady(true); });
  const { data: sub } = supabase.auth.onAuthStateChange((_e, s) => setSession(s));
  return () => sub.subscription.unsubscribe();
}, []);
```

Gate routes by segment — with `(auth)` and `(app)` route groups, redirect when
the session and the segment disagree:

```tsx
const segments = useSegments();
const router = useRouter();
useEffect(() => {
  if (!ready) return;                                  // ← without this gate you
  const inApp = segments[0] === "(app)";               //   redirect on the initial
  if (!session && inApp) router.replace("/(auth)/sign-in");   // null and bounce the
  if (session && !inApp) router.replace("/(app)");            // user through login
}, [session, ready, segments]);
```

The `ready` flag matters: `getSession()` is async, and deciding routes before
it resolves logs every user "out" for one render — the flash-of-login-screen
bug. (Current expo-router also offers `Stack.Protected` guards; the effect
above is the version that works everywhere.)

On signup, email confirmation defaults ON — the confirmation link opens on the
phone and needs the deep-link handling of §4, or turn confirmations off in
early dev (local stack: `supabase/config.toml`).

## 4. OAuth — system browser + deep link back

No redirects to `/auth/callback` pages here; the browser hands control back to
the app via its scheme (`myapp` — set in `app.json` by `expo-create-app`).

```bash
npx expo install expo-web-browser expo-auth-session
```

```ts
import * as WebBrowser from "expo-web-browser";
import { makeRedirectUri } from "expo-auth-session";

const redirectTo = makeRedirectUri();   // dev: exp://…; standalone: myapp://

export async function signInWithGoogle() {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: "google",
    options: { redirectTo, skipBrowserRedirect: true },
  });
  if (error) throw error;

  const res = await WebBrowser.openAuthSessionAsync(data.url, redirectTo);
  if (res.type !== "success") return;                        // user cancelled

  const code = new URL(res.url).searchParams.get("code");    // PKCE code, not tokens
  if (!code) throw new Error(`no code in callback: ${res.url}`);
  const { error: exchangeError } = await supabase.auth.exchangeCodeForSession(code);
  if (exchangeError) throw exchangeError;
}
```

`skipBrowserRedirect: true` is what makes this work: without it supabase-js
tries to navigate a `window` that does not exist. The `flowType: "pkce"` from
§1 is what makes the callback carry a `code` — with the default implicit flow
you get tokens in a URL *fragment*, which `openAuthSessionAsync` mangles and
you cannot exchange.

**Two things only the user can do**, say so and continue with email auth
meanwhile: configure the OAuth provider's credentials in the Supabase
dashboard, and add the redirect URLs (`myapp://**`, plus the `exp://…` dev URL
printed by `makeRedirectUri()`) to Auth → URL Configuration. A redirect URL
missing from that allowlist fails with a generic error page in the browser,
nowhere near the actual cause.

Magic links are the same shape — `signInWithOtp({ options: { emailRedirectTo:
redirectTo } })`, then the same code-exchange when the link opens the app.

## 5. Data — plug into TanStack Query

Queries stay thin wrappers; caching/refetch policy is Query's job
(`rn-state-management` §1):

```ts
useQuery({
  queryKey: ["todos", session!.user.id],
  queryFn: async () => {
    const { data, error } = await supabase.from("todos").select("*").order("created_at");
    if (error) throw error;      // throw, or Query caches an "error-shaped success"
    return data;
  },
  enabled: !!session,
});
```

`[]` with no error while data exists in Studio is the RLS-policy symptom —
`NextJs/SupabaseSetup` §5 is the fix, verbatim. Realtime works over WebSocket
unchanged; in the subscription callback, `queryClient.invalidateQueries` is the
lazy-and-correct integration.

Generated types come from the same command as the web
(`npx supabase gen types typescript --local > types/database.ts`) — regenerate
after every migration.

## Verify

1. Round trip on a **physical device**, not just the emulator — that is the
   only test of §2's networking.
2. Sign in, kill the app, relaunch: still signed in (persistence). Leave it
   backgrounded past token expiry (or shorten `jwt_expiry` locally), reopen:
   still signed in (the AppState refresh wiring).
3. Sign out lands on the login screen; a cold start while signed out shows
   login with **no flash of the app** (the `ready` gate).
4. OAuth: complete the browser round-trip on Android AND iOS — scheme handling
   differs and only device testing proves it.

## Notes

- Session storage: AsyncStorage means tokens sit unencrypted in app storage —
  standard practice and fine for most apps (app sandboxing is the boundary).
  If the threat model demands more, wrap `expo-secure-store` as the `storage`
  adapter — but it caps values at 2 KB, so use the aes-encrypt-then-store
  pattern from Supabase's RN guide, not SecureStore directly.
- `EXPO_PUBLIC_*` in a release build points at the hosted project; local URLs
  in a shipped binary are the "works on my phone" release bug. Check before
  `eas build --profile production`.
- Related: `NextJs/SupabaseSetup` (stack, SQL, RLS, types — the shared half),
  `rn-state-management` (Query wiring), `rn-build-ios-android` (release env),
  `electron-supabase-setup` (the desktop sibling of this skill).
