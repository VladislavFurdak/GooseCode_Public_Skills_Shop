# Landing Page Skill Pack

Four composable Claude Code skills for building high-converting, distinctive landing pages on **Next.js (App Router) + Vercel** — covering the full pipeline: strategy → copy → art direction → section assembly → SEO → performance → deploy.

Built from original research (July 2026) rather than copied templates: conversion canon (Unbounce Conversion Benchmark Report, NN/g eyetracking, Baymard form studies, CXL social-proof research, Julian Shapiro's handbook, Harry Dry's Marketing Examples), 2025–2026 design-trend analysis (Figma trend report, Awwwards/Lapa galleries, "AI slop" post-mortems), a survey of 20+ existing skills in the ecosystem (Anthropic's frontend-design, Hallmark, taste-skill, Impeccable, marketingskills, vercel-labs/agent-skills…), and current stack facts (Next.js 16, Tailwind v4, Motion v12, free GSAP 3.15, shadcn CLI 3.0, Vercel Flags SDK).

## Skills

| Skill | Answers | Use standalone for |
|---|---|---|
| [LandingPageBuilder](LandingPageBuilder/SKILL.md) | "Build me a landing page" — the whole thing | Full builds: intake → strategy → design → copy → code → SEO → deploy, with quality gates |
| [LandingBlocks](LandingBlocks/SKILL.md) | Which sections, in what order, built how | Adding/fixing a specific section (hero, pricing, FAQ…); Next.js/Tailwind code patterns |
| [LandingCTA](LandingCTA/SKILL.md) | What the page says | Headlines, CTA buttons & microcopy, forms, social proof, copy review with benchmarks |
| [LandingDesign](LandingDesign/SKILL.md) | How the page looks & moves | Art direction (10 named styles), color/type/motion systems, anti-"AI slop" audit |

## How they compose

```
LandingPageBuilder  (pipeline + gates)
 ├─ Phase 1 strategy   → references/page-recipes.md   (section order by goal)
 ├─ Phase 2 design     → LandingDesign                (direction, tokens, slop gates)
 ├─ Phase 3 copy       → LandingCTA                   (headlines, CTA, forms, proof)
 ├─ Phase 5 build      → LandingBlocks                (catalog + code-patterns.md)
 └─ Phases 6–8         → references/nextjs-vercel.md  (meta/OG/JSON-LD, forms, perf, deploy)
```

Each skill also triggers on its own — a request like "rewrite the headline" hits LandingCTA directly without the full pipeline.

## Opinions this pack holds

- **One goal per page** (attention ratio → 1:1), CTA repeated only with fresh proof.
- **Copy before components**; all strings live in `content/copy.ts`, never inline in JSX.
- **Commit to one named art direction**; a written token brief (`design/design-system.md`) precedes any page code; 15 anti-slop gates run before delivery.
- **Server Components by default**, client islands only with a reason; hero image gets `priority` and never hides behind an entry animation.
- **Never invent proof** — missing logos/testimonials/stats become explicit `TODO` placeholders.
- Quality bar: LCP ≤ 2.5s · INP ≤ 200ms · CLS ≤ 0.1 (mobile), WCAG AA contrast, `prefers-reduced-motion` honored.

## Stack snapshot (mid-2026)

Next.js 16.x (App Router, Turbopack, PPR-stable) · Tailwind CSS v4 (`@theme`, OKLCH) · Motion v12 (`motion/react`) · GSAP 3.15+ (fully free) · shadcn/ui CLI 3.0 registries · block sources: Tailark, Magic UI, Motion Primitives, Origin UI, HyperUI (MIT) · Vercel: Analytics, Speed Insights, Flags SDK, `ImageResponse` OG.

Versions are pinned in `LandingPageBuilder/references/nextjs-vercel.md` — the patterns outlive the version numbers.
