---
name: nextjs-setup-shadcn
description: Use right after shadcn init, before building screens — or on a Tailwind v4 App Router app that looks wrong for no obvious reason. Fixes six failures a green build misses — serif fallback fonts, content under a fixed sidebar, ignored theme tokens, hydration mismatches, vanishing navigation.
ParentSkill: shadcn
---

# Why shadcn setups render broken

Each failure here is **a contract that nothing verifies** — usually between two
files, and #1 to #4 are entirely **silent**: `shadcn init` succeeds,
`tsc --noEmit` and `next build` pass, the dev server returns 200. The toolchain
cannot see any of them. That is the whole reason this skill exists — check the
contracts explicitly, because nothing else will.

The last two are not silent, they are *unread*: #5 prints in the browser console
where a build log never shows it, and #6 only appears once the window is narrow
enough — a state nobody tests on a 1280px developer screen.

Scaffold with `nextjs-create-app` first; it applies break #1 only.

| # | Contract between | What breaks it | Symptom |
|---|---|---|---|
| 1 | `globals.css` ↔ itself | `--font-sans: var(--font-sans)` | whole app in Times New Roman |
| 2 | `globals.css` ↔ `layout.tsx` | font variable mounted on `<body>`, token computed at `:root` | still Times New Roman after fixing #1 |
| 3 | sidebar ↔ main | `<aside>` is `fixed`, content reserves no space | content slides under the menu |
| 4 | theme ↔ pages | pages hardcode `text-gray-900` | theme installed but invisible, dark mode dead |
| 5 | server HTML ↔ client DOM | extension attributes, `typeof window`, `Date.now()`, bad nesting | console hydration error, subtree re-rendered from scratch |
| 6 | sidebar ↔ viewport | `hidden lg:flex` with nothing replacing it | below `lg` the navigation is gone, not collapsed |

## 1 — The font token references itself

`shadcn init` overwrites the correct line `create-next-app` wrote:

```css
--font-sans: var(--font-geist-sans);   /* create-next-app — correct */
--font-sans: var(--font-sans);         /* shadcn init — a cycle */
```

A custom property referencing itself is a cycle, and CSS makes every property in
a cycle *guaranteed-invalid*. Three things die at once: the variable is empty,
`.font-sans` is never generated at all, and `--default-font-family` (which
preflight puts on `html`) is poisoned. Nothing sets a font, so the browser falls
to its default serif.

```bash
node -e "const f='src/app/globals.css',fs=require('fs');let s=fs.readFileSync(f,'utf8');const b=s;s=s.replace(/--font-sans:\s*var\(--font-sans\)\s*;/,'--font-sans: var(--font-geist-sans);');if(s===b)console.log('[fonts] no self-reference found');else{fs.writeFileSync(f,s);console.log('[fonts] repaired')}"
```

## 2 — The font variable sits one level below the token

**Fixing #1 alone can change nothing on screen.** The compiled CSS is correct and
the page is still serif — this is the step everyone misses.

`next/font` does not define a variable globally; it emits a class you must mount:

```css
.geist_xxx_variable { --font-geist-sans: "Geist", "Geist Fallback"; }
```

The scaffold mounts it on `<body>`, so the variable exists from `<body>` down.
But Tailwind declares `--font-sans` at `:root` — one level **above**.

A custom property is computed **where it is declared**, and only the computed
result inherits. `--font-sans` is computed at `:root`, so `var(--font-geist-sans)`
resolves *there*, finds nothing, and goes invalid. That invalid value then
inherits into the whole tree — including elements inside `<body>` that do have
the variable. They receive the broken value from above and never look it up again.

```tsx
- <html lang="en">
-   <body className={`${geistSans.variable} ${geistMono.variable} antialiased`}>
+ <html lang="en" className={`${geistSans.variable} ${geistMono.variable}`}>
+   <body className="antialiased">
```

**Do not instead move `@apply font-sans` from `html` to `body`.** It is the
intuitive fix and it fails: the problem is where `--font-sans` is *computed*, not
where it is *applied*, and that is `:root` either way.

## 3 — Fixed sidebar, no offset on the content

`position: fixed` removes the sidebar from flow, so it reserves **no space** in
the flex row. The content starts at x=0 with full width, and the sidebar paints
on top of it:

```
<aside>  fixed   x: 0 → 256      (hidden lg:flex lg:w-64 lg:fixed lg:inset-y-0)
<main>   static  x: 0 → 1265     margin-left: 0, padding-left: 0
```

Compensate by exactly the sidebar width, **at the same breakpoint** the sidebar
becomes fixed — the offset must not apply where the sidebar is `hidden`:

```tsx
- <main className="flex-1 overflow-auto">
+ <main className="flex-1 overflow-auto lg:pl-64">      /* lg:pl-64 = lg:w-64 */
```

Any `fixed`/`absolute` sidebar needs this. A sidebar left as a normal flex child
does not — that is the alternative fix.

## 4 — Pages ignore the theme they installed

A correct install still looks unthemed if screens paint around it. Never write
raw palette for surfaces, text or borders:

| Instead of | Use |
|---|---|
| `text-gray-900` | `text-foreground` |
| `text-gray-500`, `text-gray-400` | `text-muted-foreground` |
| `bg-white` | `bg-background`, `bg-card` on raised surfaces |
| `bg-gray-50`, `bg-gray-100` | `bg-muted`, `bg-secondary` |
| `border-gray-200` | `border-border` |
| `bg-blue-600 text-white` | `bg-primary text-primary-foreground` |
| `bg-red-50 text-red-700` | `bg-destructive/10 text-destructive` |

Two measured consequences:

- **Dark mode is dead code.** The theme ships a 33-variable `.dark` block; markup
  pinned to `text-gray-900` on `bg-white` ignores all of it. Toggling `.dark` on
  such a page moved nothing — contrast 20.6 → 20.6.
- **The greys clash.** Tailwind `gray-*` is blue-tinted, a `neutral` shadcn base
  is achromatic: `text-gray-500` is `lab(47.78 -0.39 -10.03)` against
  `--muted-foreground` at `lab(48.50 0 0)`. Same lightness, different hue, side by
  side — reads as muddiness with no obvious cause.

```bash
grep -rnE "(text|bg|border)-(gray|slate|zinc|neutral|red|blue|green|yellow)-[0-9]{2,3}" \
  src --include=*.tsx --include=*.ts | grep -v "components/ui"
```

Include `.ts` — status colour maps hide in `constants.ts`. Hits inside
`components/ui` are the CLI's and not yours. A healthy project returns nothing;
the project this came from returned 92 lines across seven files while
`components/ui` was clean — the exact signature of a theme installed and then
ignored.

## 5 — Hydration mismatch on `<body>`

```
A tree hydrated but some attributes of the server rendered HTML didn't match
the client properties. This won't be patched up.

  <body className="min-h-full flex flex-col"
-   data-new-gr-c-s-check-loaded="14.1311.0"
-   data-gr-ext-installed=""
  >
```

Read the diff markers before doing anything. `-` on attributes **nobody in the
codebase wrote** means they arrived after the server rendered: a browser
extension (these two are Grammarly) edited the DOM before React hydrated. There
is no bug in the app, and no amount of restructuring will remove them — they come
from the user's browser, not the build.

That specific class is what `suppressHydrationWarning` is for:

```tsx
<html lang="en" suppressHydrationWarning>
  <body className="min-h-full flex flex-col" suppressHydrationWarning>
```

It applies **one level deep only** — it silences the element it sits on, not the
tree below. That is precisely why it is safe on `<html>`/`<body>`, where
extensions inject, and dishonest anywhere else: on a component that genuinely
mismatches it hides your bug instead of fixing it, and React still discards the
server HTML for that subtree.

Also put it on `<html>` when a theme provider writes the `.dark` class from
`localStorage` — the server cannot know the stored preference, so the class
legitimately differs on the first paint.

**Every other cause in that error message is your bug. Do not suppress those:**

| Cause | Why it mismatches | Fix |
|---|---|---|
| `typeof window !== "undefined"` in render | Server takes one branch, client the other | Render the server branch, switch in `useEffect` |
| `Date.now()`, `Math.random()` | New value on each call | Compute once on the server and pass down, or render after mount |
| `toLocaleDateString()` with no locale | Server locale/timezone ≠ the user's | Pass an explicit locale **and** `timeZone`, or format client-side |
| Live external data | Server and client fetch different snapshots | Send the server's snapshot as the initial value |
| Invalid nesting | The browser silently *moves* nodes, so the DOM stops matching | Fix the markup |

Invalid nesting is the one this stack makes easy. `asChild` merges props onto
whatever child you pass, so the tag you end up with is not the tag you read:

```tsx
<p>
  <Button asChild><div>Save</div></Button>   {/* <div> inside <p> — the browser hoists it out */}
</p>
```

`<p>` may not contain block elements, `<a>` may not contain `<a>`, `<button>` may
not contain `<button>`, and `<tr>` accepts only `<td>`/`<th>`. Each one is
rearranged by the parser before React sees it, which is why it reads as a
hydration error rather than a layout bug.

## 6 — The menu vanishes as the window narrows

`hidden lg:flex` on the sidebar is the shadcn/Tailwind default shape, and on its
own it means that below `lg` the navigation is **gone** — not collapsed, not
behind a button. Nothing replaces it, so at 1023px the user cannot reach any
other page of the app.

Losing navigation is not a cosmetic regression; a fixed sidebar with no offset
(#3) merely overlaps, this removes the only route out of the current screen.

The desktop sidebar must be paired with a narrow-width trigger that opens the
same links:

```tsx
{/* desktop: unchanged */}
<aside className="hidden lg:flex lg:w-64 lg:fixed lg:inset-y-0">
  <Nav />
</aside>

{/* below lg: the same Nav, reachable */}
<Sheet>
  <SheetTrigger asChild className="lg:hidden">
    <Button variant="ghost" size="icon" aria-label="Open menu">
      <MenuIcon />
    </Button>
  </SheetTrigger>
  <SheetContent side="left" className="w-64 p-0">
    <SheetTitle className="sr-only">Navigation</SheetTitle>
    <Nav />
  </SheetContent>
</Sheet>
```

Three things that are easy to get wrong here:

- **One `Nav`, rendered twice.** Two copies of the link list drift apart — a route
  added to one and not the other is invisible until someone opens the app on a
  phone.
- **`SheetTitle` is required** even when hidden. Every shadcn overlay needs a
  title for screen readers; use `sr-only` rather than omitting it.
- **The breakpoints must agree.** The trigger's `lg:hidden`, the sidebar's
  `hidden lg:flex` and the content's `lg:pl-64` from #3 are one decision written
  three times. Change one and you get either two menus at once or none.

Check it by narrowing the window, not by reading the classes:

```bash
grep -rn "hidden lg:flex\|hidden md:flex" src --include=*.tsx
```

For each hit, confirm a `Sheet`/`Drawer` trigger exists in the same layout. Then
open the app at 375px wide and reach every top-level destination. If you cannot,
the menu is missing however correct the markup looks.

## Verify

Static — cross-checks `globals.css` against `layout.tsx`, catching #1 and #2. Use
the heredoc: the same script via `node -e "..."` gets mangled by shell quoting and
falsely fails on a healthy project.

```bash
node <<'EOF'
const fs = require('fs');
const css = fs.readFileSync('src/app/globals.css', 'utf8');
const lay = fs.readFileSync('src/app/layout.tsx', 'utf8');
const m = css.match(/--font-sans:\s*var\((--[\w-]+)\)/);
if (!m) throw new Error('globals.css: no "--font-sans: var(...)" declaration');
const ref = m[1];
if (ref === '--font-sans') throw new Error('BREAK 1: --font-sans references itself');
const declared = [...lay.matchAll(/variable:\s*["']([^"']+)["']/g)].map(x => x[1]);
if (!declared.includes(ref))
  throw new Error(`layout.tsx declares [${declared}] but globals.css needs ${ref}`);
if (!/className=/.test((lay.match(/<html[^>]*>/) || [''])[0]))
  throw new Error('BREAK 2: font variable class is not on <html>');
console.log(`OK: --font-sans -> ${ref}; declared in layout.tsx; mounted on <html>`);
EOF
```

Runtime — catches what static cannot: whether the font actually *rendered*, and
#3. Run in the browser at a desktop width.

```js
const problems = [];
const f = getComputedStyle(document.body).fontFamily;
if (/Times|^serif/i.test(f)) problems.push('font fell back to serif: ' + f);
const aside = document.querySelector('aside'), main = document.querySelector('main');
if (aside && main && getComputedStyle(aside).position === 'fixed'
    && getComputedStyle(aside).display !== 'none') {
  const edge = aside.getBoundingClientRect().right;
  const xs = [...main.querySelectorAll('h1,h2,p,table,[data-slot="card"]')]
    .map(e => e.getBoundingClientRect().left).filter(x => x > 0);
  if (xs.length && Math.min(...xs) < edge)
    problems.push(`content starts at ${Math.round(Math.min(...xs))}px, under sidebar ending at ${Math.round(edge)}px`);
}
problems.length ? problems : 'OK';
```

Check both breakpoints — the offset in #3 is breakpoint-scoped, and mobile must
show no offset and no horizontal overflow.

Narrow — catches #6, which only exists below the breakpoint. Resize to 375px
first; at desktop width it always passes.

```js
const problems = [];
const aside = document.querySelector('aside');
const asideHidden = !aside || getComputedStyle(aside).display === 'none';
// A trigger is anything that opens the nav: shadcn's Sheet marks it with
// data-slot, a hand-rolled one is a visible button with an aria-label.
const trigger = document.querySelector(
  '[data-slot="sheet-trigger"], [data-slot="drawer-trigger"], header button[aria-label*="enu" i]',
);
const triggerVisible = !!trigger && getComputedStyle(trigger).display !== 'none';
if (asideHidden && !triggerVisible)
  problems.push('navigation is unreachable at this width: sidebar hidden and no menu trigger');
if (document.documentElement.scrollWidth > window.innerWidth + 1)
  problems.push(`horizontal overflow: ${document.documentElement.scrollWidth}px in ${window.innerWidth}px`);
problems.length ? problems : 'OK';
```

Then open the menu and confirm the links inside are the same set the desktop
sidebar shows. The check above proves a trigger exists, not that it leads
anywhere.

The console must also be clean of the hydration error in #5 — it appears on load,
not on interaction, so read it right after the first paint.

`tsc --noEmit` and `next build` must pass too, but neither detects anything above.
Never report the setup done on a green build alone.

## Typography

shadcn's own site, measured: headings **600** with **negative tracking** (`-2.4px`
at 48px), labels 500, body 400. `font-bold` (700) at default tracking on
everything is what still looks unfinished next to the real thing, even once the
font is correct.

- Headings → `font-semibold` + `tracking-tight`
- Labels, table headers, nav → `font-medium`

## Notes

- Do this before building screens. Afterwards means editing every file already
  written.
- A shadcndesign theme (`shadcn-select-theme`) **masks break #1** by declaring a
  concrete font later in the cascade — remove the theme and the serif returns.
  Repair the wiring anyway. Break #2 survives themes untouched.
- Serif can only mean an empty `--font-sans`. The stack is `"Geist", "Geist
  Fallback"` and the fallback is `local(Arial)`, so a failed font *download*
  yields Arial, never Times. Seeing Times after a verified fix means a stale page
  — hard-reload before debugging further.
- A hydration mismatch whose diff names only a `<body>` attribute — `cz-shortcut-listen`
  (ColorZilla), `grammarly-*`, `data-lt-*` — is a browser extension, not your code.
  Confirm by checking the attribute is absent from `curl` output and from the
  source, or by loading the page in a private window. Silence it with
  `suppressHydrationWarning` on `<body>`; it covers that one element's attributes
  and text only, never its descendants, so it cannot hide a real mismatch.
- Related: `nextjs-create-app` (scaffold + break #1), `shadcn-select-theme`
  (palette choice + loading that theme's font).
