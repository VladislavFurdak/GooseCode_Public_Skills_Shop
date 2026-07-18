---
name: shadcn-select-theme
description: Chooses and installs a shadcn/ui theme that fits the product being built, from the twelve free shadcndesign themes, and loads the font each one needs. Use once a shadcn project exists and no visual direction has been given. Not for projects with an established brand or palette.
ParentSkill: shadcn
---

# Choosing a shadcn theme

Requires an existing shadcn project (`components.json` present) and shadcn CLI
3.0+. Themes are CSS-only, so they work with both the Radix UI and Base UI
versions.

## First: should you pick a theme at all?

**Do not** if the project already has a visual direction — a brand palette, a
logo whose colours the UI should echo, a Figma file, a screenshot to match, or an
existing app to stay consistent with. Adopt that instead. A theme applied over an
existing identity looks broken.

**Do not** if the user named a colour or mood ("make it blue", "keep it neutral").
Follow what they said. If one of these themes happens to match, use it; if none
does, edit the CSS variables directly rather than installing the nearest miss.

Pick a theme when the project is new, the request gave no visual direction, and
the default neutral shadcn palette would leave it looking unfinished.

## How to choose

Judge the product, not your taste. In order:

**1. What is it, and who opens it every day?** An invoicing tool for accountants
and a recipe app for teenagers should not look alike. Read the domain from the
request, the entity names, and any copy already written.

**2. How dense is the screen?** This constrains more than anything else. A
dashboard of tables, filters and numbers needs a restrained theme — a vivid
accent repeated across 200 rows becomes noise and destroys the hierarchy the data
depends on. Reserve loud palettes for marketing pages, landing pages, and
consumer apps with a lot of whitespace.

**3. Does it need a dark mode, or is it dark-first?** Some themes below are built
dark. Do not put them on a product that ships light-only.

**4. Does the theme's SHAPE fit?** Several themes commit to more than a hue —
zero border radius, offset "sticker" shadows, a serif face. Those read far louder
than colour and will fight a conventional business UI. They are the right answer
when the product wants personality, and the wrong one when it wants to disappear.

**5. Still torn between two?** Ask the user once, naming both and what each would
say about the product. Do not silently guess on something this visible. Do not
ask if one clearly fits.

## The twelve themes

| Theme | Palette | Font | Commits to | Reads as |
|---|---|---|---|---|
| `theme-purple` | rich violet | Rubik | tinted shadows | creative tools, premium SaaS |
| `theme-honey` | amber / gold | Nunito Sans | — | warm, welcoming, approachable |
| `theme-soft-orange` | orange, blue undertone | Onest | — | balanced, modern product UI |
| `theme-night-wind` | teal / cyan | Inter | strong light **and** dark | calm, professional, health, data |
| `theme-icecream` | pink / salmon pastel | Instrument Sans | — | soft consumer, food, beauty, kids |
| `theme-citrus` | lime green | DM Sans | offset shadows | fresh, energetic |
| `theme-fresh-lime` | bright lime | Host Grotesk | bold offset shadows | youthful, high-energy |
| `theme-sketchpad` | sketch style | Geist Mono | offset shadows, hand-drawn | playful, notebook, dev-facing |
| `theme-fast-red` | high-contrast red | Anta | **zero border radius**, sharp | urgent, bold, sport, performance |
| `theme-orbiter` | orange-red | TASA Orbiter (custom) | stark contrast | fiery, technical, space |
| `theme-neon-forest` | neon green | Inter | **dark-first**, deep forest bg | cyber, dev tools, monitoring |
| `theme-soulslike` | muted brown-olive | Domine (**serif**) | **dark**, medieval | games, editorial, lore-heavy |

Quick shortlists:

- **Business / admin / dashboard / analytics** → `night-wind`, `soft-orange`, `purple`
- **Marketing or landing page** → `purple`, `citrus`, `fresh-lime`, `orbiter`
- **Consumer, food, wellness, kids** → `icecream`, `honey`, `citrus`
- **Developer tooling, monitoring, logs** → `neon-forest`, `sketchpad`, `night-wind`
- **Games, entertainment, editorial** → `soulslike`, `orbiter`, `fast-red`
- **Finance, legal, healthcare, government** → `night-wind` or `soft-orange`. Nothing louder; these audiences read novelty as untrustworthy.

## Install

The command prompts for confirmation and **overwrites existing CSS variables and
components**, so pass `--yes` — without it the CLI blocks forever waiting on
stdin. The `@shadcndesign` namespace resolves without any `components.json`
registry configuration; shadcn CLI 3.0+ is required.

Install the theme **before** building screens: it overwrites theme variables, and
any hand-tuned colours will be lost.

Each theme is two operations, never one. The install and the font import belong
together — run the command, then immediately add the import to `layout.tsx`:

| Theme | Install command | Then import in `layout.tsx` |
|---|---|---|
| Purple | `npx shadcn@latest add @shadcndesign/theme-purple --yes` | `import { Rubik } from "next/font/google"` |
| Honey | `npx shadcn@latest add @shadcndesign/theme-honey --yes` | `import { Nunito_Sans } from "next/font/google"` |
| Sketchpad | `npx shadcn@latest add @shadcndesign/theme-sketchpad --yes` | `import { Geist_Mono } from "next/font/google"` |
| Soft Orange | `npx shadcn@latest add @shadcndesign/theme-soft-orange --yes` | `import { Onest } from "next/font/google"` |
| Fast Red | `npx shadcn@latest add @shadcndesign/theme-fast-red --yes` | `import { Anta } from "next/font/google"` |
| Night Wind | `npx shadcn@latest add @shadcndesign/theme-night-wind --yes` | `import { Inter } from "next/font/google"` |
| Citrus | `npx shadcn@latest add @shadcndesign/theme-citrus --yes` | `import { DM_Sans } from "next/font/google"` |
| Soulslike | `npx shadcn@latest add @shadcndesign/theme-soulslike --yes` | `import { Domine } from "next/font/google"` |
| Fresh Lime | `npx shadcn@latest add @shadcndesign/theme-fresh-lime --yes` | `import { Host_Grotesk } from "next/font/google"` |
| Orbiter | `npx shadcn@latest add @shadcndesign/theme-orbiter --yes` | `import { TASA_Orbiter } from "next/font/google"` |
| Neon Forest | `npx shadcn@latest add @shadcndesign/theme-neon-forest --yes` | `import { Inter } from "next/font/google"` |
| Icecream | `npx shadcn@latest add @shadcndesign/theme-icecream --yes` | `import { Instrument_Sans } from "next/font/google"` |

All twelve families are on Google Fonts (verified), and `next/font/google` names
them with underscores for spaces. The catalogue `next/font` ships is a snapshot
taken at each Next.js release, so a recently-added family may be missing from an
older Next — `TASA Orbiter` is the newest here and the most likely to be absent.
If an import fails to resolve, fall back to a `<link>` to Google Fonts in the
layout, or `next/font/local` with the files.

## Load the font — REQUIRED, and nothing will tell you

**The theme sets the font but does not load it.** Verified on a clean Next.js 16
project with `theme-purple`:

```css
/* globals.css after installing the theme */
--font-sans: Rubik, sans-serif;
```
```tsx
// layout.tsx — untouched by the install, still the scaffold default
import { Geist, Geist_Mono } from "next/font/google";
```

Rubik appears nowhere else in the project. The declared font simply is not there,
so the browser falls through to generic `sans-serif`, and the theme's typography
— half of what you chose it for — is silently absent. `next build` passes. Only a
human looking at the screen catches it.

After installing, read the font out of the CSS and wire it up:

```bash
grep -n -- "--font-sans\|--font-heading\|--font-mono" src/app/globals.css
```

Then in `src/app/layout.tsx`, load that family and expose it under the variable
name the CSS expects:

```tsx
import { Rubik } from "next/font/google";

const sans = Rubik({ subsets: ["latin"], variable: "--font-sans" });

// on <html>:
className={`${sans.variable} antialiased`}
```

Match the variable name to whatever `globals.css` actually references — if it
declares `--font-sans` directly with a family name, either replace that
declaration with `var(--font-sans)` from the loaded font, or name the loaded
font's variable to match. The two must agree; that is the whole failure mode.

The exact import for every theme is in the table above.

## Verify

```bash
npx tsc --noEmit
npx next build
```

Then look at a rendered page and confirm the font actually changed. A build that
passes proves nothing here — every failure mode in this skill is silent.

If the type still looks like the default sans, the font variable in `layout.tsx`
does not match the one in `globals.css`.

## Notes

- A theme replaces the palette wholesale. Install one, not two.
- Do not hand-edit theme variables straight after installing — decide the theme
  first, then adjust. Re-running an install discards your edits.
- Related: `nextjs-create-app` documents a separate font bug where `shadcn init`
  leaves `--font-sans` referencing itself, rendering the app in Times New Roman.
  Installing a theme happens to mask it, because the theme declares a concrete
  value later in the cascade. Removing the theme brings it back.
