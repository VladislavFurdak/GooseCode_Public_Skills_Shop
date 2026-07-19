---
name: supabase-setup
description: Use before writing any data layer or auth on a Next.js App Router app. Sets up Supabase with @supabase/ssr, the Next.js client — browser client, server client and session middleware, plus the local Docker stack, SQL migrations, generated types and how to deploy those migrations to a hosted project. No ORM. A plain supabase-js createClient is the wrong client here and breaks auth across the server/client boundary. Start local; a hosted project needs an account only the user can create.
---

# Supabase on the Next.js App Router

Two decisions shape everything below.

**Use `@supabase/ssr`, not a plain `supabase-js` client.** On the App Router a
request is handled in three different places — browser, server, middleware — and
the session lives in cookies. One shared `createClient()` cannot reach those
cookies from all three, so auth appears to work in the browser and the user is
anonymous in every server component. `@supabase/ssr` is the Next.js client;
`supabase-js` comes along as its dependency and is not what you construct.

**Develop against the local stack, deploy to a hosted project.** Local Supabase
is the full thing — Postgres, PostgREST, Auth, Storage, Studio — running in
Docker with no account, no browser and no keys to ask anyone for. That is what
makes this buildable in one uninterrupted run. Creating the hosted project is the
user's job, not yours (§7).

Nothing here compiles native code.

## 1 — Install

```bash
npm i @supabase/supabase-js @supabase/ssr      # timeout: 300000
npm i -D supabase                              # the CLI, or call it with npx
```

`@supabase/ssr` is not optional on the App Router. Sessions live in cookies, and
a single `createClient` cannot read or write them across the server/client
boundary — auth appears to work in the browser and then the user is anonymous in
every server component.

## 2 — Start the local stack

Requires Docker running. If Docker is not available, skip to §7 and ask the user
for a hosted project — there is no third option.

```bash
npx supabase init      # timeout: 300000 — writes supabase/config.toml
npx supabase start     # timeout: 600000 — FIRST run pulls ~3 GB of images
```

The first `start` downloads about a dozen images and takes minutes; later starts
take seconds. Give it the full 600000 ms — a timeout here leaves a half-pulled
stack whose next error looks nothing like a timeout.

When it finishes it prints the credentials. Fixed local ports, the same on every
machine:

| What | Where |
|---|---|
| API (use as `SUPABASE_URL`) | `http://127.0.0.1:54321` |
| Postgres (direct) | `postgresql://postgres:postgres@127.0.0.1:54322/postgres` |
| Studio (browse data in a UI) | `http://127.0.0.1:54323` |
| Mailpit (catches auth emails) | `http://127.0.0.1:54324` |

`npx supabase status` reprints all of it, including `ANON_KEY` and
`SERVICE_ROLE_KEY`. Read the keys from there rather than hardcoding them — they
are stack-generated, and pasting a stale one produces an authentication error
that reads like a bug in your query.

Useful to know: local containers are named `supabase_db_<project>`,
`supabase_rest_<project>` and so on, where `<project>` is `project_id` in
`supabase/config.toml`. That is how you tell two local stacks apart, and why
code running *inside* the Docker network addresses the database by container
name rather than by `127.0.0.1`.

## 3 — Environment

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=<ANON_KEY from `supabase status`>
SUPABASE_SERVICE_ROLE_KEY=<SERVICE_ROLE_KEY from `supabase status`>
```

**The anon key is public by design** — it goes to the browser, and Row Level
Security is what protects the data. **The service-role key bypasses RLS
entirely.** Never put it in a `NEXT_PUBLIC_*` variable, never import it into a
client component, never log it. It belongs only in server code: route handlers,
server actions, server components.

## 4 — Clients

Three files, because the App Router has three places a request can be handled.
Do not collapse them into one shared client — each needs different cookie access.

**Browser** — client components:

```ts
// lib/supabase/client.ts
import { createBrowserClient } from "@supabase/ssr";
import type { Database } from "@/types/database";

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  );
}
```

**Server** — server components, route handlers, server actions:

```ts
// lib/supabase/server.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";
import type { Database } from "@/types/database";

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options),
            );
          } catch {
            // Called from a Server Component, which cannot set cookies. Safe to
            // ignore ONLY because the middleware below refreshes the session.
          }
        },
      },
    },
  );
}
```

The empty `catch` is deliberate and is Supabase's documented pattern: a Server
Component may read cookies but never write them, so a refresh attempted there
must fall through silently — and the middleware is what actually persists it.
Without the middleware this swallow becomes a session that never refreshes.

**Middleware** — refreshes the session on every request:

```ts
// middleware.ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options),
          );
        },
      },
    },
  );

  // Refreshes the token. Must run before any redirect logic.
  await supabase.auth.getUser();

  return supabaseResponse;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"],
};
```

**Return `supabaseResponse` itself.** If you build a different response, copy
`supabaseResponse.cookies` onto it first — dropping them logs the user out at
random, and the symptom appears pages later, nowhere near the cause.

### getUser, never getSession, on the server

```ts
const { data: { user } } = await supabase.auth.getUser();
```

`getSession()` reads the cookie and believes it. On the server that cookie is
attacker-supplied data. `getUser()` revalidates the token with the Auth server,
so it is the only one that may gate access to anything. Use `getSession()` in
the browser only, where the session was established locally.

Pass the `Database` generic from §6 to every client. Without it every row is
`any`, and a typo in a column name becomes a runtime `null` rather than a
compile error.

For server-only work that must bypass RLS, build a separate service-role client
in a file no client component imports — see §3.

## 5 — Schema and migrations

Schema is SQL in versioned files. There is no ORM to declare models in.

```bash
npx supabase migration new create_users      # writes supabase/migrations/<timestamp>_create_users.sql
```

This works offline — it only creates the file. Write the SQL yourself:

```sql
create table public.users (
  id bigint generated always as identity primary key,
  full_name text,
  phone varchar(256),
  created_at timestamptz not null default now()
);

alter table public.users enable row level security;

create policy "read users" on public.users
  for select to anon using (true);
```

**Enable RLS on every table, then add the policies you actually want.** A table
with RLS on and no policy returns an empty array to the anon key — no error, no
warning, just `[]`. That silent empty result is the single most common way to
lose time here: the query is correct, the data exists, and Studio shows it.

Apply to the local database:

```bash
npx supabase migration up      # applies PENDING migrations — non-destructive
```

Use `npx supabase db reset` only when you deliberately want the local database
rebuilt from scratch: it **drops the data** and replays every migration. Never
run it against anything you would miss.

## 6 — Generated types

```bash
npx supabase gen types typescript --local > types/database.ts
```

Re-run it after every migration. The generator has four sources — `--local`,
`--linked`, `--db-url` and `--project-id` — so the same command works against a
hosted project once linked.

## 7 — Going remote: this needs the user

**You cannot do this step.** Creating a Supabase account and project requires a
signup that only the user can complete. Do not attempt it, and do not invent a
connection string to move on.

When the app is ready for a hosted database, tell the user plainly — something
like:

> To deploy, I need a hosted Supabase project. Register at
> https://supabase.com/dashboard, create a project, then send me its **project
> ref** (in the dashboard URL) and the **database password** you set. Until then
> I keep building against the local stack — nothing is lost, the same migrations
> apply to both.

Then keep working locally rather than stopping. The local stack and the hosted
project run the same Postgres and the same migrations; switching over is §8, and
it is a five-minute step at the end, not a prerequisite at the start.

## 8 — Deploying migrations to a hosted project

Once the user supplies the project ref:

```bash
npx supabase login                          # opens a browser — the user does this
npx supabase link --project-ref <ref>       # timeout: 300000
npx supabase db push                        # timeout: 300000 — applies local migrations to the remote database
```

`db push` sends the migration files that the remote has not recorded yet. It
does not push data, and it does not reset anything.

Then point the app at the hosted project by swapping the two environment
variables — Dashboard → Project Settings → API gives the project URL and anon
key. Nothing in the code changes.

Order matters: `link` before `push`, and never `db reset` against a linked
production project. If `push` reports migrations the remote does not know about,
stop and read the list rather than forcing it.

## Traps

| Symptom | Cause | Fix |
|---|---|---|
| `select` returns `[]`, data visible in Studio | RLS on, no policy for the anon role | add a `create policy … for select` |
| logged in in the browser, anonymous in server components | one plain `createClient` instead of the three in §4 | use `@supabase/ssr` |
| users logged out at random | middleware returned a response without `supabaseResponse.cookies` | return `supabaseResponse`, or copy its cookies over |
| session never refreshes | middleware missing, so the Server Component `catch` swallows every refresh | add `middleware.ts` |
| every row typed `any` | client created without the `Database` generic | pass it, and regenerate types after each migration |
| auth error that looks like a query bug | a stale hardcoded key | read keys from `npx supabase status` |
| `supabase start` fails instantly | Docker not running | start Docker, or go to §7 |
| CLI says the stack is not running | you are in the wrong directory | the CLI keys off `supabase/config.toml`; each project is its own stack |
| local data gone | `db reset` was run | it drops and replays by design — use `migration up` for pending changes |

## Verify

A green `next build` proves nothing about the data layer — it compiles fine
against a database that was never reachable. Prove a round trip:

```ts
const { data, error } = await supabase.from("users").select("id, full_name");
if (error) throw error;
console.log(data);
```

An empty array with no error is not success — check §5 for the RLS policy before
concluding the table is empty. Confirm the same rows in Studio at
`http://127.0.0.1:54323`.

## Notes

- **Timeouts: 300000 ms for installs and CLI commands, 600000 ms for the first
  `supabase start`.** The default 60 s cuts them off mid-download.
- Commit `supabase/migrations/` and `supabase/config.toml`. Never commit
  `.env.local` or any service-role key.
- `@supabase/supabase-js` warns that **Node 20 and below are deprecated**; move
  to Node 22+ when you can. Everything here still runs on 20.
- Verified 2026-07-19 on Windows, Node 20.19.1, supabase CLI 2.109.1,
  @supabase/supabase-js 2.109.0, @supabase/ssr 0.12.3: install timing, `init`,
  `start` (full pull and boot, credentials as listed), `migration new` offline,
  `gen types` sources, the typed client compiling, and the `getAll`/`setAll`
  cookie interface read from the package's own types — the older
  `get`/`set`/`remove` shape still exists but is superseded. The remote flow in
  §8 follows Supabase's documentation; it was not run here, since it needs an
  account.
- Related: `nextjs-create-app` (scaffold), `nextjs-setup-shadcn` (UI contracts).
