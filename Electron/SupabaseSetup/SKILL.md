---
name: electron-supabase-setup
description: Supabase in an Electron app — the desktop threat model (anon key only, service_role never ships), a renderer client whose session persists through main-process safeStorage instead of localStorage, OAuth via the system browser returning through a custom protocol deep link, and offline behavior a desktop app owes its users. Use before writing data or auth code. Schema/RLS/migrations are shared with NextJs/SupabaseSetup — follow that skill for the database half.
---

# Supabase on the desktop

The database half — local stack, migrations, RLS, generated types — is
`NextJs/SupabaseSetup` §2/§5/§6 unchanged, and unlike mobile, a desktop app
in dev reaches `http://127.0.0.1:54321` directly (it IS the same machine —
none of `rn-supabase-setup` §2's networking pain applies). What changes on
desktop is trust, storage, and how OAuth gets back into the app.

**The threat model sentence:** an Electron app ships to the user's disk,
where anything in it can be extracted — so the app may contain only the URL
and the anon key, and RLS is the entire authorization model. There is no
"server side" inside the app; `service_role` must never appear in main OR
renderer (main feels server-like; it is not — it ships too). Privileged
operations belong in Edge Functions or a real backend.

## 1. Client — in the renderer, with IPC-backed storage

The renderer is a browser; `@supabase/supabase-js` runs there natively
(realtime included). What must NOT be default is session storage:
localStorage is banned for anything that matters
(`electron-state-management`), and an auth token is the most
that-matters data in the app. Give the client a storage adapter that rides
IPC into main's safeStorage:

```ts
// src/renderer/lib/supabase.ts
import { createClient } from "@supabase/supabase-js";
import type { Database } from "../../shared/types/database";

const authStorage = {
  getItem: (key: string) => window.api.secureStore.get(key),        // Promise<string | null>
  setItem: (key: string, value: string) => window.api.secureStore.set(key, value),
  removeItem: (key: string) => window.api.secureStore.remove(key),
};

export const supabase = createClient<Database>(
  import.meta.env.VITE_SUPABASE_URL,
  import.meta.env.VITE_SUPABASE_ANON_KEY,
  {
    auth: {
      storage: authStorage,
      persistSession: true,
      autoRefreshToken: true,
      detectSessionInUrl: false,   // the window's URL is app://… or file://… — nothing to detect
      flowType: "pkce",            // §3 depends on this
    },
  },
);
```

The `secureStore` channels are `electron-ipc-contract` +
`electron-state-management` §4 combined: main encrypts values with
`safeStorage` into a file in `userData`. Async storage adapters are fully
supported by supabase-js. Where `isEncryptionAvailable()` is false (keyring-less
Linux), fall back to *not* persisting and asking the user to sign in per
launch — never to plaintext.

URL and anon key are public by design — bake them at build time
(`VITE_*` / bundler `define`). A packaged app must not depend on `.env`
files lying next to the binary; that is a dev-only mechanism. Before
packaging, swap to the hosted project's values — creating that hosted
project is the user's job, in exactly the words of `NextJs/SupabaseSetup` §7.

Email/password auth now works end to end, and `getSession()` in the renderer
is fine here — this is a local client trusting its own storage, not an SSR
server trusting a cookie; the `getUser()`-only rule from the Next.js skill is
about *server-side* gating, which in this architecture lives in RLS.

## 2. Route gating

Same shape as every client app: subscribe once at mount
(`supabase.auth.onAuthStateChange`), hold `session` + a `ready` flag in a
Zustand store, gate screens on it, render a neutral shell until `ready` —
the async storage adapter makes the initial `getSession()` genuinely async,
so skipping the gate flashes the login screen at users who are signed in
(`rn-supabase-setup` §3, same bug, same fix).

## 3. OAuth — system browser out, deep link back

Never render a provider's login page inside the app: an Electron window is an
embedded webview to Google, which **blocks OAuth in embedded contexts**
(`disallowed_useragent`) — and users cannot see the URL bar, which is
exactly the phishing surface the block exists for. The flow is: system
browser out, custom-protocol deep link back.

**Registering the protocol** (once):

```ts
// main — dev-mode registration on Windows needs the exe+argv form
if (process.defaultApp && process.argv[1]) {
  app.setAsDefaultProtocolClient("myapp", process.execPath, [path.resolve(process.argv[1])]);
} else {
  app.setAsDefaultProtocolClient("myapp");
}
```

plus, so *installers* register it with the OS, in electron-builder config
(the reference app ships exactly this shape):

```jsonc
"build": { "protocols": [{ "name": "MyApp", "schemes": ["myapp"] }] }
```

**Receiving the link** — two OS paths, both funneling into one handler, and
both depending on the single-instance lock from `electron-create-app` §2:

```ts
// Windows/Linux: the second launch carries the URL in argv
app.on("second-instance", (_e, argv) => {
  const url = argv.find((a) => a.startsWith("myapp://"));
  if (url) handleDeepLink(url);
  focusMainWindow();
});
// macOS: no second instance — the running app gets an event
app.on("open-url", (_e, url) => handleDeepLink(url));
```

**The flow itself:**

```ts
// renderer initiates
const { data } = await supabase.auth.signInWithOAuth({
  provider: "google",
  options: { redirectTo: "myapp://auth-callback", skipBrowserRedirect: true },
});
await window.api.auth.openExternal(data.url);   // main → shell.openExternal

// main's handleDeepLink forwards the URL to the renderer as an event;
// renderer completes:
const code = new URL(url).searchParams.get("code");
if (code) await supabase.auth.exchangeCodeForSession(code);
```

`skipBrowserRedirect: true` because there is no window to redirect;
`flowType: "pkce"` (§1) is what makes the callback carry an exchangeable
`?code=` instead of tokens in a `#fragment` — fragments never reach the app
through a protocol launch, which is the classic "callback arrives, session
never appears" dead end.

**User's half, name it and continue with email auth meanwhile:** provider
credentials configured in the Supabase dashboard, and `myapp://auth-callback`
added to Auth → URL Configuration → Redirect URLs. A missing allowlist entry
fails as a generic browser error page pointing nowhere.

Magic links: same protocol machinery via `emailRedirectTo` — works, but the
link must be opened on this machine; for desktop tools OAuth + password
covers users with less support burden.

## 4. Data layer and desktop realities

Queries plug into TanStack Query in the renderer exactly as in
`rn-supabase-setup` §5 — throw on `error`, `enabled: !!session`,
`invalidateQueries` from realtime callbacks. Regenerate types per migration
(`npx supabase gen types typescript --local > src/shared/types/database.ts` —
into `shared/`, since main-side code may query too... but keep actual querying
in ONE process; split-process writes to the same tables invite subtle races).

Desktop-specific expectations:

- **Offline is a normal state**, not an error state — laptops close and
  reopen. `retry` on queries handles blips;
  `onlineManager.setEventListener` wired to `navigator.onLine` events (works
  in the renderer) pauses/resumes cleanly. Show stale data with a quiet
  "reconnecting…" indicator, never a modal error wall on wake-from-sleep.
- **Realtime reconnects** after sleep are supabase-js's job, but the
  subscription's `status` callback is where you trigger a refetch on
  `"SUBSCRIBED"` re-entry — events missed during sleep are otherwise a
  silent gap (`invalidateQueries` on resubscribe closes it).
- `[]` with no error while Studio shows rows = missing RLS policy for `anon`/
  `authenticated` — `NextJs/SupabaseSetup` §5, verbatim.

## Verify

1. Sign in (email flow), quit fully, relaunch: signed in — and the token is
   not greppable in plaintext anywhere under `userData`
   (`electron-state-management` §Verify 5).
2. OAuth round-trip: browser opens, consent, app un-minimizes and is signed
   in — tested on Windows AND (at minimum by protocol-launch simulation) on
   the other shipping OSes; deep-link plumbing is per-OS code.
3. `myapp://auth-callback?code=garbage` launched by hand (e.g. `start
   myapp://…` on Windows): app surfaces a sane error, no crash — this input
   is attacker-reachable from any webpage.
4. Kill the network mid-session: UI degrades to stale+indicator; restore:
   data refetches without restart.
5. RLS round-trip per `NextJs/SupabaseSetup` §Verify.

## Notes

- Packaged-build check before release: the baked URL points at the hosted
  project, not `127.0.0.1` — the desktop twin of the RN release bug.
- Deep links only register properly on Linux/Windows once the app is
  *installed* (the `.desktop` file / registry entry come from the
  installer) — dev-mode protocol tests on Linux may need a manual
  `.desktop` entry; don't burn time "fixing" code the installer will fix.
- Related: `NextJs/SupabaseSetup` (database half),
  `electron-state-management` §4 (the secure store §1 rides),
  `electron-ipc-contract` (channels), `electron-build-multiplatform`
  (`protocols` config ships in the installer),
  `rn-supabase-setup` (mobile twin — same flows, different transport).
