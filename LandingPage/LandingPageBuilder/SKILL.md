---
name: landing-page-builder
description: End-to-end pipeline for building a complete, high-converting landing page on Next.js (App Router) deployed to Vercel — strategy, copy, design, section assembly, SEO/OG, performance, deploy. Use for ANY request, in any language, to create, build, or redo a landing page, promo page, product page, waitlist page, one-pager, or marketing site. Even if the user only says "make a website for my product", start here. Not for multi-page apps, dashboards, or blogs.
---

# Landing Page Builder — Next.js + Vercel

Build landing pages where strategy, copy, design, and code are one artifact, not three handoffs. This skill is the orchestrator: it owns the pipeline and the quality gates, and delegates depth to three sibling skills — `landing-cta` (copy & conversion), `landing-design` (art direction), `landing-blocks` (section catalog). If those skills are unavailable, this pipeline still works standalone.

References:
- [references/page-recipes.md](references/page-recipes.md) — section orders by page goal (SaaS, waitlist, lead-gen, app, ecommerce, event)
- [references/nextjs-vercel.md](references/nextjs-vercel.md) — stack specifics: scaffold, fonts, metadata/OG, JSON-LD, forms via Server Actions, analytics, A/B, performance, deploy

## Stack (default, mid-2026)

Next.js 16 App Router (Turbopack default; 15.5 is maintenance LTS) · TypeScript · Tailwind CSS v4 (CSS-first `@theme`) · React Server Components everywhere, `"use client"` islands only for interactivity · Motion v12 (`motion/react`) for reveals · `next/font` + `next/image` · Deploy on Vercel. shadcn/ui optional — pull marketing blocks from Tailark/Magic UI/Motion Primitives when they fit, restyled to the design system.

Quality bars, non-negotiable: LCP ≤ 2.5s · INP ≤ 200ms · CLS ≤ 0.1 (mobile) · WCAG AA contrast · `prefers-reduced-motion` honored · one primary conversion goal per page · passes the `landing-design` slop gates.

## Pipeline

Work through phases in order. Each phase ends with a named artifact; don't write page code before phases 0–3 produce their artifacts — code-first is how generic template pages happen.

### Phase 0 — Intake

Ask (or, when working autonomously, infer and STATE the assumptions in one block before proceeding):

1. Product & the one action a visitor must take (signup / waitlist / demo / buy / download)?
2. Audience & where traffic comes from (cold ads / launch post / SEO / retargeting)?
3. Top 3 objections a skeptical visitor has?
4. Proof inventory: logos, numbers, testimonials, screenshots actually available? (Never invent proof — placeholder slots are marked `TODO` for the user.)
5. Brand: existing colors/fonts/tone? Plus the 4 vibe questions from `landing-design` (place/object, one emotion, two influences, never-mistaken-for).
6. Pricing shown? Locales needed (e.g. RU + EN)?

**Artifact:** a 10-line brief at the top of your working notes.

### Phase 1 — Strategy

- Place the visitor on the awareness scale (unaware → most-aware; see `landing-cta`) — it sets page length and framework.
- Pick the page recipe from [references/page-recipes.md](references/page-recipes.md) and adapt: list final section order with one-line job per section.
- Message map: core promise, mechanism, objection→block mapping, proof→block mapping.

**Artifact:** section plan (ordered list, each with its job + the objection it answers).

### Phase 2 — Design brief

Run the `landing-design` process: pick ONE art direction, produce the token brief (OKLCH palette, type pairing, surfaces, motion language, signature element, hero ASCII wireframe), self-critique against the slop gates, then persist to `design/design-system.md`.

**Artifact:** `design/design-system.md`.

### Phase 3 — Copy

Run `landing-cta`: headline variants (pick one, keep two alternates in comments), subheadline, CTA + microcopy, then section-by-section copy following the message map. Copy is written BEFORE components so components serve the words, not vice versa. Real, specific, founder-voiced; Flesch-Kincaid ≤ grade 7.

**Artifact:** `content/copy.md` (or inline in a `content.ts` data file).

### Phase 4 — Scaffold

Follow [references/nextjs-vercel.md](references/nextjs-vercel.md) §Scaffold. Summary:

```bash
npx create-next-app@latest <name> --typescript --tailwind --app --turbopack --no-src-dir --import-alias "@/*"
cd <name> && npm i motion
```

Project shape:

```
app/
  layout.tsx        # fonts, metadata base, Analytics
  page.tsx          # assembles sections in recipe order
  globals.css       # @theme tokens from design-system.md
  opengraph-image.tsx  sitemap.ts  robots.ts  icon.svg
  actions.ts        # Server Actions (waitlist/contact)
components/
  sections/         # one file per block: hero.tsx, logos.tsx, …
  reveal.tsx  sticky-cta.tsx  …
content/copy.ts     # ALL copy as data — no strings inside JSX
design/design-system.md
```

Wire the design tokens into `globals.css` `@theme` immediately — building sections against `--color-accent` etc. means restyles are one-file changes.

### Phase 5 — Build sections

Top-down in recipe order, using `landing-blocks` (+ its `code-patterns.md`). Rules:

- Server Components by default; a `"use client"` island needs a reason (toggle, accordion-with-state, counter, sticky CTA, motion).
- Hero first and best — ~50% of design effort; `priority` on the LCP image; hero never animates in from `opacity: 0`.
- Every imported library block gets restyled to the tokens before commit.
- Real content from `content/copy.ts`; missing assets become clearly-marked placeholders (`TODO: real logo strip`), never fake logos/testimonials/stats.

### Phase 6 — SEO, meta, wiring

From [references/nextjs-vercel.md](references/nextjs-vercel.md): Metadata API (title template, description, canonical, `metadataBase`), `opengraph-image.tsx` (design it — it's part of the 50% hero effort), `sitemap.ts` + `robots.ts`, JSON-LD (Organization + SoftwareApplication/Product; FAQPage if FAQ exists), favicon set, `<Analytics/>` + `<SpeedInsights/>`, form Server Action with validation + honeypot.

### Phase 7 — Performance & accessibility pass

Checklist from [references/nextjs-vercel.md](references/nextjs-vercel.md) §Performance. Highlights: fonts via `next/font` (≤2 families + mono), below-fold images lazy (default), heavy embeds behind `next/dynamic`, third-party scripts via `@next/third-parties` or `strategy="lazyOnload"`, reserved dimensions everywhere (CLS), keyboard walk to the CTA, focus rings, contrast check, reduced-motion check. Run Lighthouse (mobile) — investigate anything red before shipping.

### Phase 8 — Deploy & measure

Vercel: `vercel` CLI or git-push → preview URL → promote. Enable Web Analytics + Speed Insights. Track CTA clicks and form submits as events. A/B tests via Flags SDK precompute pattern when the user wants experiments. Note: Vercel Hobby tier is non-commercial; commercial landings belong on Pro.

### Phase 9 — Self-audit (gate before "done")

1. **5-second test**: fresh eyes on the hero — what's sold, for whom, what's the next step?
2. **Slop gates** from `landing-design` — all 15.
3. **Conversion checklist** from `landing-cta` (CTA copy, microcopy, proof placement, form rules).
4. **Blocks checklist** from `landing-blocks` (cadence, mobile, variety).
5. Lab CWV + a11y from Phase 7.

Report results honestly — including what's placeholder and what failed.

## Iteration verbs

After v1, map feedback to targeted moves instead of full rebuilds: **bolder / quieter** (design dials ±2), **polish** (hover/focus/spacing micro-pass), **tighten** (copy cut pass), **rearrange** (section order vs recipe), **audit** (rerun Phase 9, punch list only).

## Deliverable format

Ship: running page + `design/design-system.md` + `content/copy.ts` + a short report — key decisions (direction, recipe, headline choice + alternates), TODO list of placeholder assets the user must supply, deploy URL or exact deploy command, and the Phase 9 audit summary.
