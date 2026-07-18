---
name: nextjs-setup-prisma
description: Use before writing any data layer on a Next.js app that needs persistence — or when Prisma is already installed and seeding fails with "Cannot find module generated/prisma", PrismaClientInitializationError, or "url is no longer supported in schema files". Prisma 7 removed the datasource URL and made a driver adapter mandatory; the obvious setup is now unreachable on Windows.
---

# Why Prisma setups spiral

Prisma 7 changed three things at once, and each one breaks a step that used to
work by default:

| What changed | What it breaks |
|---|---|
| `url` removed from the `datasource` block | every schema written from memory fails validation |
| a driver adapter is now **mandatory** | `new PrismaClient()` with no argument throws at runtime |
| `migrate dev` no longer runs `generate` | the client is missing, so the import fails |

Alone, each is a one-line fix. Together they read as one broken install, because
fixing any one of them surfaces the next — and the error messages point at
symptoms, not at the version change underneath.

Two agent runs on the same brief, measured: one spent **14** database commands
getting out, the other **46** and about forty minutes, ending with
`npm install prisma@6` and `rm prisma.config.ts` — it escaped by going back a
major version rather than by solving anything. In that run `npx tsx
prisma/seed.ts` ran **13 times and produced 7 different errors**: not a loop, a
cascade, each fix genuinely revealing the next wall.

The recipe below, measured clean: **26 seconds** from empty directory to a seeded
database, zero errors.

## Decide first: 6 or 7

**Default to 6.** It has no adapter, no config file, and its SQLite engine is a
downloaded binary — nothing compiles. Choose 7 only when something else requires
it.

The deciding fact is not preference, it is **Windows**. Prisma 7 needs a driver
adapter, and the obvious SQLite adapter needs a native module:

```
npm error command failed
npm error command ... prebuild-install || node-gyp rebuild --release
npm error prebuild-install warn install No prebuilt binaries found
npm error   (target=20.19.1 runtime=node arch=x64 platform=win32)
```

`better-sqlite3` ships **no prebuilt binary** for Node 20 on win32-x64, so npm
falls through to `node-gyp`, which needs Visual Studio Build Tools. On a machine
without them the install cannot complete at all. This is the dead end both
measured runs wandered into.

If you need 7, use **`@prisma/adapter-libsql`** instead — pure JS, installs
clean, verified working below.

## Path A — Prisma 6 (default)

```bash
npm install prisma@6 @prisma/client@6 tsx    # timeout: 300000
npx prisma init --datasource-provider sqlite
rm -f prisma.config.ts                       # 6.19 scaffolds a 7-style project
```

`prisma init` is misleading even on 6: it writes the **new** generator and a
`prisma.config.ts`. Version 6 still accepts the classic shape, so overwrite the
schema — do not keep what it scaffolded:

```prisma
generator client {
  provider = "prisma-client-js"     // NOT "prisma-client" — no output path
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")    // legal on 6, rejected on 7
}
```

`.env` already holds `DATABASE_URL="file:./dev.db"` from `init`. Leave it.

```bash
npx prisma migrate dev --name init   # timeout: 300000 — runs generate too, on 6
```

The client lands in `node_modules/@prisma/client`, so the import is the plain
one and the seed needs no adapter and no `dotenv` — Prisma 6 reads `.env` itself:

```ts
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();
```

Ignore the warning `init` writes into `.env` about variables not being loaded
automatically. It describes the 7 config flow. On 6 it is false, and following it
adds a `dotenv` dependency you do not need.

## Path B — Prisma 7 (only if required)

```bash
npm install prisma @prisma/client @prisma/adapter-libsql @libsql/client dotenv tsx   # timeout: 300000
npx prisma init --datasource-provider sqlite
```

**Keep `prisma.config.ts`.** On 7 it is where the connection URL lives, and
deleting it is what starts the spiral:

```ts
import "dotenv/config";
import { defineConfig } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: { path: "prisma/migrations" },
  datasource: { url: process.env["DATABASE_URL"] },
});
```

The datasource block in the schema carries **no `url`** — that is the change, not
a mistake in the scaffold:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../generated/prisma"
}

datasource db {
  provider = "sqlite"
}
```

```bash
npx prisma migrate dev --name init   # timeout: 300000
npx prisma generate                  # REQUIRED on 7 — migrate no longer does it
```

Three details that each cost a round trip if guessed:

- The client is at the **custom output path**, not in `node_modules`:
  `import { PrismaClient } from "./generated/prisma/client"` — path relative to
  the importing file, and note the trailing `/client`.
- The adapter export is **`PrismaLibSql`**. Not `PrismaLibSQL`. Writing the
  acronym in caps yields `TypeError: PrismaLibSQL is not a constructor`, which
  reads like a version mismatch and is only a capital `Q`.
- `import "dotenv/config"` is needed in scripts run outside the Prisma CLI.

```ts
import "dotenv/config";
import { PrismaClient } from "./generated/prisma/client";
import { PrismaLibSql } from "@prisma/adapter-libsql";

const adapter = new PrismaLibSql({ url: process.env["DATABASE_URL"]! });
const prisma = new PrismaClient({ adapter });
```

## Reading the error you already have

Match the message before changing anything — each of these means one specific
thing, and three of them are the same root cause seen from different angles.

| Message | Cause | Fix |
|---|---|---|
| `P1012 ... property 'url' is no longer supported` | schema written for 6, CLI is 7 | move the URL to `prisma.config.ts`, or pin to 6 |
| `Cannot find module '../generated/prisma'` | `generate` never ran, or output path moved | run `npx prisma generate`; check the generator's `output` |
| `PrismaClientInitializationError` | 7 client constructed with no adapter | pass `adapter`, or pin to 6 |
| `PrismaClientConstructorValidationError` | option that 7 removed (e.g. `datasources`) | drop it; the URL is config-side now |
| `PrismaLibSQL is not a constructor` | export is `PrismaLibSql` | fix the casing |
| `prebuild-install ... No prebuilt binaries` | `better-sqlite3` has no Windows prebuild | use `@prisma/adapter-libsql`, or Path A |
| `Cannot find module 'prisma/config'` | `prisma.config.ts` present, `prisma` not installed | finish the install before running the CLI |

## Verify

Do not report the data layer done on a green `next build` — it compiles fine
against a client that cannot connect. Prove a round trip instead:

```bash
npx tsx prisma/seed.ts
```

A seed that writes and then reads back is the check. Counting after an upsert
proves the schema migrated, the client generated, the URL resolved and the
adapter connected — the four things that break separately:

```ts
await prisma.user.upsert({
  where: { email: "admin@example.com" },
  update: {},
  create: { email: "admin@example.com", name: "Admin" },
});
console.log("seeded:", await prisma.user.count());
```

Re-run it once. `upsert` is idempotent, so a second run must print the same
count — if it climbs, the `where` is not on a unique column and the seed is
duplicating rows on every deploy.

## Notes

- **Give installs 300000 ms.** `npm install prisma` pulls an engine binary; the
  default 60 s kills it midway and leaves `node_modules` in a state whose next
  error looks nothing like a timeout.
- Pick the version **before** the first install. Switching after migrations exist
  means deleting `prisma/migrations` and the `.db` file too, since the two majors
  disagree about where the URL lives.
- Pinning to 6 is a deliberate trade, not an oversight: it buys a setup with no
  native compilation on Windows. Revisit when a prebuilt SQLite adapter exists
  for the target Node version.
- Verified 2026-07-19 on Windows, Node 20.19.1, against prisma 6.19.3 and 7.8.0.
  Both paths were run end to end from an empty directory.
- Related: `nextjs-create-app` (scaffold), `nextjs-setup-shadcn` (UI contracts).
