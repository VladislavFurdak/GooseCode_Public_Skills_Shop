---
name: electron-ipc-contract
description: The typed IPC backbone every Electron feature hangs off — one shared channel map, a preload bridge exposing a namespaced API, Zod validation in main, and unsubscribe-returning event listeners. Use immediately after scaffolding, before the first feature. Prevents the untyped ipcRenderer.send spaghetti that makes Electron apps unmaintainable by feature three. Pattern lifted from a production app (GooseCode) running ~50 channels this way.
---

# One contract between three processes

IPC is the API boundary of the whole app — main is the server, the renderer
is the client, and the preload is the SDK. Untyped, ad-hoc
`ipcRenderer.send("do-thing", stuff)` calls scattered through components rot
fast: channel-name typos fail silently, payload shapes drift, and nothing
tells you which features touch which capabilities. The fix is one mechanical
pattern applied everywhere, so adding a feature is adding entries — never
inventing structure.

## 1. Shared channel map — the single source of truth

```ts
// src/shared/ipc.ts — imported by ALL THREE processes; keep it dependency-free
export const IPC = {
  settingsGet: "settings:get",
  settingsSet: "settings:set",
  settingsChanged: "settings:changed",   // main → renderer event
  filesOpen: "files:open",
  appInfo: "app:info",
} as const;

export type IpcChannel = (typeof IPC)[keyof typeof IPC];
```

Channel names follow `domain:verb`. Every channel that exists in the app is a
line in this file — grep-able, diff-reviewable, and a typo becomes a
TypeScript error instead of a handler that never fires. The production
reference runs its entire surface (sessions, agent streaming, auth, updater,
file dialogs — ~50 channels) out of exactly this shape.

Alongside it, the API the renderer will see:

```ts
// src/shared/api.ts
export interface AppApi {
  settings: {
    get(): Promise<Settings>;
    set(patch: Partial<Settings>): Promise<Settings>;
    onChanged(listener: (s: Settings) => void): () => void;   // returns unsubscribe
  };
  files: { open(path: string): Promise<{ ok: boolean; error?: string }> };
  app: { info(): Promise<{ version: string; platform: NodeJS.Platform }> };
}
```

## 2. Preload — build the API, expose it, expose nothing else

```ts
// src/preload/index.ts
import { contextBridge, ipcRenderer } from "electron";
import { IPC } from "../shared/ipc";
import type { AppApi } from "../shared/api";

function on<T>(channel: string, listener: (payload: T) => void): () => void {
  const wrapped = (_e: unknown, payload: T) => listener(payload);
  ipcRenderer.on(channel, wrapped);
  return () => ipcRenderer.removeListener(channel, wrapped);
}

const api: AppApi = {
  settings: {
    get: () => ipcRenderer.invoke(IPC.settingsGet),
    set: (patch) => ipcRenderer.invoke(IPC.settingsSet, patch),
    onChanged: (l) => on(IPC.settingsChanged, l),
  },
  files: { open: (path) => ipcRenderer.invoke(IPC.filesOpen, path) },
  app: { info: () => ipcRenderer.invoke(IPC.appInfo) },
};

contextBridge.exposeInMainWorld("api", api);
```

Two hard rules:

- **Never expose `ipcRenderer` itself** (nor a generic
  `send(channel, data)` passthrough). That hands the renderer every channel
  that will ever exist, deleting the security boundary the preload is for.
  Only concrete, named functions cross the bridge.
- **Every subscription returns its unsubscribe.** The `on()` helper makes it
  automatic. Without it, React strict-mode double-mounts and HMR stack
  duplicate listeners, and events start firing twice — a bug that looks like
  a backend problem and is actually a leak.

Type the renderer's view once:

```ts
// src/renderer/global.d.ts
import type { AppApi } from "../shared/api";
declare global { interface Window { api: AppApi } }
```

Renderer code now calls `window.api.settings.get()` with full types and zero
`"electron"` imports (`electron-create-app` §Notes).

## 3. Main — handle, validate, return result objects

The bridge is typed, but types are compile-time only — **payloads arrive in
main as untrusted `unknown`**. The threat model: a compromised renderer (one
XSS in rendered content) can invoke any exposed channel with any payload.
Validate at the boundary with Zod, exactly like a web server validates a
request body:

```ts
// src/main/ipc.ts
import { ipcMain, shell } from "electron";
import { z } from "zod";
import { IPC } from "../shared/ipc";

const SettingsPatch = z.object({
  theme: z.enum(["light", "dark", "system"]).optional(),
  locale: z.string().max(20).optional(),
}).strict();

export function registerIpc(deps: { settings: SettingsStore }) {
  ipcMain.handle(IPC.settingsGet, () => deps.settings.get());

  ipcMain.handle(IPC.settingsSet, (_e, raw: unknown) => {
    const patch = SettingsPatch.parse(raw);          // throws → rejects the invoke
    return deps.settings.set(patch);
  });

  ipcMain.handle(IPC.filesOpen, async (_e, raw: unknown) => {
    const rel = z.string().min(1).parse(raw);
    const abs = resolveInsideWorkspace(rel);          // §4
    if (!abs) return { ok: false, error: "Path is outside the workspace." };
    const error = await shell.openPath(abs);
    return error ? { ok: false, error } : { ok: true };
  });
}
```

Conventions that keep this maintainable:

- `invoke`/`handle` for everything request-shaped. Reserve raw
  `send`/`on` for main→renderer pushes only.
- **Expected failures are return values** (`{ ok: false, error }`), not
  thrown errors — a throw crosses IPC as an opaque
  `Error invoking remote method…` string with the stack stripped, useless in
  the UI. Throw only for programmer errors (validation), return everything
  the UI must explain.
- All registrations happen in one `registerIpc()` called from main's startup
  — not sprinkled through modules — so the handler list and the channel map
  stay reviewable side by side.

## 4. Payload rules learned in production

- **Validate paths against a root.** Any renderer-supplied file path gets
  resolved and prefix-checked against the workspace/allowed dir before fs
  use — the reference app rejects with exactly
  `"Path is outside the workspace."`. Path traversal via IPC is the desktop
  equivalent of an unchecked route param.
- **Serializable data only.** IPC uses structured clone: no functions, no
  class instances (they arrive as plain objects), Dates survive but arrive
  as Dates only via structured clone — keep payloads JSON-plain so Zod can
  own the contract.
- **Big binaries don't belong in chat.** For images the reference app returns
  `data:` URIs with a hard size cap (25 MB) because the sandboxed renderer's
  CSP forbids `file://`; beyond that, write a temp file and pass the path
  back. Never stream megabytes through repeated events.
- **Streams are event sequences.** Long operations (an LLM turn, a build)
  send `domain:event` pushes carrying `{ sessionId, delta }`-shaped payloads,
  with the renderer subscribed once at mount via the unsubscribe pattern —
  not one giant invoke that resolves minutes later with everything.

## Verify

1. `tsc --noEmit` passes with the renderer importing only `window.api` types.
2. In devtools: `window.api` exists, `window.api.settings.get()` resolves;
   `ipcRenderer` is undefined.
3. Call an exposed channel with a garbage payload from the console
   (`window.api.settings.set({ evil: 1 })` — with `.strict()` this must
   reject) — proves validation is on.
4. Mount/unmount a component subscribed to an event five times, fire the
   event once: the listener runs once (unsubscribes work).

## Notes

- The pairing with state: which channels exist for settings/state sync and
  who owns the data is `electron-state-management`; this skill is the
  transport those decisions ride on.
- When a channel needs to be *more* trusted (auth tokens, shell execution),
  also check `event.senderFrame.url` is the app's own URL before handling —
  cheap, and it blocks any iframe that ever sneaks into the renderer.
- Related: `electron-create-app` (the sandbox that shapes the preload),
  `electron-state-management`, `electron-supabase-setup` (session storage
  rides this bridge), `electron-localization` (locale changes propagate as a
  `settings:changed` event).
