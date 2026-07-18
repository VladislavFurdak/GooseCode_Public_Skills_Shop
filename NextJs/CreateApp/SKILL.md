---
name: nextjs-create-app
description: Scaffolds a Next.js App Router project that shadcn/ui installs into cleanly, and repairs the upstream `shadcn init` bug that leaves such projects rendering in Times New Roman. Use BEFORE creating a Next.js app, ahead of the shadcn skill. Not for an app that already exists.
---

# Next.js scaffold that shadcn can land on

Produces an empty, building, shadcn-ready Next.js project — and nothing else. No
components, no design system, no example pages: the shadcn skill supplies those,
and anything invented here would have to be undone before it can.

## When to use this

Use it when the task starts with "build a web app / dashboard / admin panel /
site" and no project exists yet. Stop as soon as the app builds — hand the actual
UI work to the shadcn skill.

Do **not** use it on an existing project. Re-running the scaffold over one
overwrites `globals.css` and `layout.tsx`.

## 1. Scaffold

Run from the directory that should CONTAIN the project, not from inside it:

```bash
npx create-next-app@latest <app-name> \
  --typescript --tailwind --app --src-dir --import-alias "@/*" \
  --no-eslint --use-npm --yes
```

**Give this command at least 5 minutes** (`timeout_ms: 300000`, or leave it
unset — that is the default). It downloads a registry's worth of packages, so
its runtime belongs to the network, not to the machine, and a minute is normal.

Do not shorten it. A scaffold killed mid-install is the worst outcome available
here: the directory already exists and is half-populated, so the retry fails
with *"directory contains files that could conflict"* and you are now cleaning
up instead of building. Observed three times on this exact command at 60s.

Three of these flags are load-bearing for shadcn and must not be dropped:

| Flag | Why shadcn needs it |
|---|---|
| `--src-dir` | `components.json` resolves `@/components`, `@/lib`, `@/hooks` under `src/`. Without it `shadcn add` writes files where imports cannot find them. |
| `--import-alias "@/*"` | Every file shadcn generates imports through `@/`. Without the alias the project does not type-check. |
| `--app` | shadcn's generated components are React Server Components by default (`"rsc": true`). |

`--tailwind` and `--typescript` are also required — shadcn writes its theme into
the Tailwind CSS file and emits `.tsx`.

`--no-eslint` only keeps the scaffold non-interactive; add `--eslint` if you want
linting. It has no effect on shadcn compatibility.

Then, from inside the project, confirm the runtime and the versions you actually
got rather than assuming — Next.js majors change defaults:

```bash
node --version && npx next --version
```

## 2. Initialise shadcn

```bash
npx shadcn@latest init --defaults --yes
```

Same rule as the scaffold: **at least 5 minutes**. This step also installs
dependencies, and it is the second command that gets killed at 60s.

This writes `components.json`, `src/lib/utils.ts`, a first component, and rewrites
`src/app/globals.css` with the theme variables.

Do not hand-write `components.json`. Let the CLI produce it, then read it — the
shadcn skill reads the same file to learn the project's aliases, base library and
icon set.

## 3. Fix the font variable — REQUIRED

`shadcn init` breaks the font wiring that `create-next-app` set up correctly.
Verified on Next 16.2 / shadcn `base-nova`:

```css
/* create-next-app writes this — correct */
--font-sans: var(--font-geist-sans);

/* shadcn init replaces it with this — a self-reference */
--font-sans: var(--font-sans);
```

`layout.tsx` still exposes the loaded font as `--font-geist-sans`, so nothing
defines `--font-sans` any more. A custom property that references itself is
invalid at computed-value time, so `font-family: var(--font-sans)` becomes
`unset`, inherits, finds nothing at the root, and lands on the browser default:
**Times New Roman**. Since `globals.css` applies `font-sans` to `body`, the whole
app renders in a serif face.

It is silent — the project installs, type-checks, builds and runs. Only a human
looking at the screen catches it.

Fix it immediately after `init`:

```bash
node -e "const f='src/app/globals.css',fs=require('fs');let s=fs.readFileSync(f,'utf8');const b=s;s=s.replace(/--font-sans:\s*var\(--font-sans\)\s*;/,'--font-sans: var(--font-geist-sans);');if(s===b)console.log('[fonts] nothing to fix — check globals.css by hand');else{fs.writeFileSync(f,s);console.log('[fonts] repaired --font-sans')}"
```

Then confirm, and confirm the name matches what `layout.tsx` actually defines:

```bash
grep -n -- "--font-sans\|--font-mono" src/app/globals.css
grep -n "variable:" src/app/layout.tsx
```

`--font-sans` must point at the same variable `layout.tsx` declares. If a future
scaffold names it something other than `--font-geist-sans`, use that name.
`--font-heading: var(--font-sans)` needs no separate fix — it resolves through
the line you just repaired.

## 4. Verify before handing over

```bash
npx tsc --noEmit
npx next build
```

Both must pass. Do not start building UI on a project that does not yet build —
a scaffold error found later looks like a bug in your own code.

If `next build` fails on Windows with `path length ... exceeds max length of
filesystem`, the project directory is nested too deeply. Move it nearer the drive
root; it is not a code problem.

## What not to do here

- **Do not add a design system, tokens, or component recipes.** shadcn brings its
  own, and two systems in one project look broken. This skill deliberately leaves
  the visual language empty for shadcn to fill.
- **Do not hand-write components shadcn provides** (button, card, dialog, table,
  form fields). Install them with `shadcn add` so they match the project's theme
  and stay updatable.
- **Do not deliver a React app as a single HTML file with CDN `<script>` tags.**
- **Do not edit `globals.css` beyond the font repair above** until shadcn has been
  initialised — `init` rewrites the file and your edits are lost.

## Hand-off

The project is ready when `next build` passes and `components.json` exists. From
that point the shadcn skill owns component choice, composition and theming; run
`npx shadcn@latest info --json` to give it the project context it expects.
