---
name: electron-state-management
description: Where state lives in an Electron app — main process owns persistence (atomic JSON settings store in userData, validated with Zod), the renderer holds a Zustand cache hydrated over IPC and updated by push events, secrets go through safeStorage, and localStorage is banned for anything that matters. Use before the first feature that remembers anything. Prevents split-brain state between processes and data loss on cache clears.
---

# Main owns the truth; the renderer holds a cache

In a web app the server owns durable state and the browser caches it. Electron
has the same split hiding inside one binary: **main is the server** (fs
access, outlives renderer reloads, shared by all windows), the renderer is a
client. Every state bug specific to Electron — settings that revert, two
windows disagreeing, data vanishing after an update — is this boundary drawn
wrong.

| State | Owner | Mechanism |
|---|---|---|
| User settings (theme, locale, workspace path) | main | JSON store in `userData` (§1) |
| Documents / sessions / anything user-created | main | files in `userData`, one module per domain |
| Secrets (tokens, API keys) | main | `safeStorage` (§4) |
| Ephemeral UI (open panels, selection, drafts) | renderer | Zustand, not persisted |
| Server data (if a backend exists) | neither | TanStack Query in the renderer (`electron-supabase-setup`) |

**localStorage holds nothing important, ever.** It is scoped to the session
partition, invisible to main, and wiped by "clear cache"-style actions or
partition changes across updates. Settings that survive in dev and vanish for
real users after an update trace back here.

## 1. The settings store — ~60 lines, no dependency

`electron-store` is a fine off-the-shelf answer; the hand-rolled version is
small enough to own and easier to bend (the reference app rolls its own).
The three properties that matter, whichever you pick:

```ts
// src/main/settings.ts
import { app } from "electron";
import fs from "node:fs";
import path from "node:path";
import { z } from "zod";

const SettingsSchema = z.object({
  theme: z.enum(["light", "dark", "system"]).default("system"),
  locale: z.string().default("system"),
  windowBounds: z.object({ x: z.number(), y: z.number(), width: z.number(), height: z.number() }).nullable().default(null),
}).default({});
export type Settings = z.infer<typeof SettingsSchema>;

export class SettingsStore {
  private file = path.join(app.getPath("userData"), "settings.json");
  private value: Settings;
  private listeners = new Set<(s: Settings) => void>();

  constructor() {
    let raw: unknown = {};
    try { raw = JSON.parse(fs.readFileSync(this.file, "utf8")); } catch {}
    const parsed = SettingsSchema.safeParse(raw);
    this.value = parsed.success ? parsed.data : SettingsSchema.parse({});
    // ↑ corrupt or outdated file → defaults, never a crash at startup
  }

  get(): Settings { return this.value; }

  set(patch: Partial<Settings>): Settings {
    this.value = SettingsSchema.parse({ ...this.value, ...patch });
    const tmp = this.file + ".tmp";                     // atomic write: a crash
    fs.writeFileSync(tmp, JSON.stringify(this.value, null, 2));   // mid-write can
    fs.renameSync(tmp, this.file);                      // never half-corrupt the real file
    for (const l of this.listeners) l(this.value);
    return this.value;
  }

  onChange(l: (s: Settings) => void): () => void { this.listeners.add(l); return () => this.listeners.delete(l); }
}
```

The three properties: **schema-validated load with default fallback** (a
corrupt settings file must cost the settings, not the app), **atomic
tmp-then-rename write**, **change notification** feeding §2. Schema evolution
comes free: new fields get `.default()`s, old files parse forward.

If the app's data directory should be shared with a sibling CLI or differ
from `productName`, pin it explicitly — **before any store is constructed**:

```ts
app.setPath("userData", path.join(app.getPath("appData"), "my-app"));
```

The reference app does exactly this so its desktop and CLI share sessions.
Calling it after something already read the default path splits data across
two directories — a bug users report as random data loss.

## 2. Sync to the renderer — hydrate once, then push

Channels ride `electron-ipc-contract` (`settingsGet` / `settingsSet` /
`settingsChanged`). Main's side wires the store's `onChange` to a broadcast:

```ts
settingsStore.onChange((s) => {
  for (const win of BrowserWindow.getAllWindows())
    win.webContents.send(IPC.settingsChanged, s);
});
```

Renderer keeps a Zustand mirror, hydrated once and then event-driven:

```ts
// src/renderer/stores/settings.ts
export const useSettings = create<{ settings: Settings | null }>(() => ({ settings: null }));

export async function initSettings(): Promise<void> {   // call once at app mount
  useSettings.setState({ settings: await window.api.settings.get() });
  window.api.settings.onChanged((s) => useSettings.setState({ settings: s }));
}

export function updateSettings(patch: Partial<Settings>): void {
  void window.api.settings.set(patch);   // no local set — the changed event round-trips
}
```

The one-way loop (renderer asks → main writes → main pushes → every window
updates) is what makes multi-window and reload correctness automatic. Do
**not** optimistically `setState` in `updateSettings` — with the round-trip
push it double-fires; without validation it can disagree with what main
actually accepted. The round-trip on localhost IPC is imperceptible.

`settings: null` until hydration is deliberate — boot decisions (theme class,
locale, initial route) must wait for it, the same gate as
`rn-state-management` §3's hydration rule. Render a neutral shell, not a
default-theme flash.

## 3. Ephemeral renderer state — plain Zustand

Selection, open/closed panels, in-progress composer text: normal Zustand
stores with selectors (`useUi((s) => s.sidebarOpen)`), **no persist
middleware** — persistence on this side of the boundary is how localStorage
sneaks back in. If a piece of "UI" state should survive restart (sidebar
width, last-open tab), that is the tell it belongs in the settings schema —
promote it rather than persisting locally.

Domain data (documents, sessions) stays in main behind list/get/save channels;
the renderer requests and subscribes, holding results in component state or a
store that is happily lost on reload. The reference app runs its entire
session list this way, including reconciling external writes (its CLI shares
the store) by re-reading and pushing a `changed` event — a shape worth copying
if anything else ever writes the data dir.

## 4. Secrets — safeStorage, nothing else

API keys and tokens go through Electron's `safeStorage`
(OS keychain / DPAPI / kwallet-backed):

```ts
// main, alongside the settings store but a separate file — secrets.json.enc
const enc = safeStorage.encryptString(token).toString("base64");
const dec = safeStorage.isEncryptionAvailable() ? safeStorage.decryptString(Buffer.from(enc, "base64")) : null;
```

Plaintext tokens in `settings.json` are readable by anything with the user's
file access; `safeStorage` binds them to the OS user account. Renderer never
sees stored secrets unless a feature genuinely requires it — it asks main to
*use* the secret (`supabase session restore`, `api call`), not to *return*
it, wherever possible. Check `isEncryptionAvailable()` — on some Linux
setups without a keyring it is false; degrade to asking the user to re-login
rather than silently writing plaintext.

## Verify

1. Change theme → relaunch: persists. Corrupt `settings.json` by hand
   (write `{"theme": 42}`) → relaunch: app opens with defaults, file heals on
   next write, no crash.
2. Open two windows, change a setting in one: the other updates without
   focus/reload.
3. Reload the renderer (Ctrl+R) mid-session: settings and domain data return
   identical — nothing lived only in renderer memory.
4. `grep -rn "localStorage" src/renderer` → nothing (or only a commented
   grave marker).
5. Search the userData directory for a known token value: not found in
   plaintext.

## Notes

- One writer per file: main. If a CLI or second process must write the same
  data, copy the reference app's watch-and-reconcile pattern instead of
  letting both write blind.
- Window bounds restore (from `electron-modern-design` §6) is just another
  settings field — validate against current displays on restore.
- Related: `electron-ipc-contract` (the transport), `electron-localization`
  + `electron-modern-design` (locale/theme live here),
  `electron-supabase-setup` (its session adapter is §4 applied),
  `rn-state-management` (the same philosophy with the mobile tools).
