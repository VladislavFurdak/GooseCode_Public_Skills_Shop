---
name: landing-design
description: Create a distinctive, intentional visual design for landing and marketing pages — art direction, color/typography/motion systems, and an anti-"AI slop" quality gate. Use whenever the user asks to design, restyle, or redesign a landing page, hero, or marketing site, asks for a creative, beautiful, or premium look and feel, complains a page looks generic or AI-generated, or before writing any landing page code. Not for app dashboards or admin UI.
---

# Landing Design — Art Direction & Anti-Slop

Produce landing pages that look like a boutique studio made them, not like the average of every SaaS template. The enemy is convergence: purple gradients on dark, default shadcn cards, Inter everywhere, centered symmetry, fade-in on everything. A design is working when a visitor could not mistake it for any other company's page.

## Modes

Pick the mode from the request:

- **Build** — new design from a brief (default; full process below).
- **Audit** — score an existing page against the slop gates and design systems; deliver a punch list, change nothing.
- **Redesign** — keep content and information architecture, replace the visual system (run the full process, skip content work).
- **Study** — extract "design DNA" from a site the user admires (palette, type, spacing, motion, signature moves) into a portable `design-system.md` they can reuse.

## Process: brief → critique → build

Never start writing page code from a cold prompt. Slop happens when the model reaches for its statistical default; a written brief forces decisions first.

### 1. Intake (4 questions + context)

Ask (or infer and state your assumptions if the user is unavailable):

1. If this brand were a real-world **place or object**, what would it be? (a Muji store? a machine shop? a private bank lobby?)
2. What is the **one emotion** a visitor must feel in the first 3 seconds? (trust / awe / play / calm / power)
3. Name **two unexpected influences** to collide (e.g. "Swiss timetables × terminal UIs", "botanical prints × fintech").
4. What must this page **never be mistaken for**? (name the cliché to run from)

Plus: industry, audience, light or dark preference, existing brand assets.

### 2. Pass 1 — written design brief (tokens before pixels)

Emit a compact brief, and persist it to `design/design-system.md` in the project so later sessions and other skills stay consistent:

- **Direction**: one named art direction from [references/directions.md](references/directions.md) — read that file and commit to exactly one. Steal at most one accent idea from a second direction.
- **Palette**: 4–6 named colors in OKLCH with roles (`--bg`, `--surface`, `--text`, `--muted`, `--accent`, optional `--accent-2` for illustration only). Follow 60-30-10: ~60% background, 30% surfaces/tints, 10% accent. One saturated accent reserved for CTAs.
- **Typography**: display face + body face (+ mono for labels/numbers). Never Inter alone — that's the #1 AI tell. Distinctive pairings that work: Bricolage Grotesque + Inter, Space Grotesk + Work Sans, Fraunces + Inter, Instrument Serif + Geist, Playfair Display + Raleway; Inter-alternatives: Sora, Outfit, Plus Jakarta Sans, Onest, Figtree; mono accents: JetBrains Mono, Geist Mono, IBM Plex Mono. Hand-tuned scale (12/14/16/18/20/24/30/36/48/60/72/96px), body 16–18px, display line-height 0.95–1.1, body 1.5–1.7, tracking −0.02em on large display type, fluid hero via `clamp(2.5rem, 1rem + 6vw, 6rem)`.
- **Surfaces**: border width/color, radius scale, shadow style — these three carry most of a direction's personality (compare `4px 4px 0 #000` vs `0 1px 2px rgba(0,0,0,.06)`).
- **Motion language**: 2–3 sentences (e.g. "slow confident 500ms rises, no bounce, marquee for logos, one scroll-linked hero effect").
- **Signature element**: exactly ONE bold risk — a bespoke hero interaction, unusual nav, textured background, custom cursor, oversized typographic moment. One risk, everything else quiet.
- **Hero wireframe**: 5–10 line ASCII sketch of the above-the-fold layout (forces a layout decision; default to left-aligned or asymmetric, not centered-everything).

### 3. Pass 2 — self-critique

Before building, attack your own brief: "Which of these choices would an image model have generated anyway?" Check every slop gate below; if the brief trips one, revise. Only then write code.

### 4. Build

Spend effort where attention goes: **~50% of design effort on the hero + OG image** — 57% of viewing time is above the fold. Craft the small proofs of intentionality: `::selection` color, focus rings matching the accent, favicon, hover states on everything interactive, real content (never lorem, never fake logos).

## Color rules

- Define tokens in OKLCH (`oklch(0.72 0.19 145)`) — perceptually uniform, Tailwind v4 native, P3-capable.
- Tint grays toward the brand hue; never pure `#808080` neutrals. Design in grayscale first, add color last.
- Dark mode: base `#0a0a0a`–`#121212` (never `#000`), text `#e8e8e8` not `#fff`, desaturate accents vs light mode, elevation via lighter surface tiers + 1px borders (`rgba(255,255,255,0.06–0.12)`), not shadows.
- Contrast: WCAG AA — 4.5:1 body, 3:1 large text — including text over gradients and glass.
- Gradients only textured (add grain/noise layer) and contained (hero only). An untextured purple→blue gradient is the canonical AI marker.

## Motion rules

Motion budget: every animation earns its place by directing attention, communicating state, or expressing the brand — blanket fade-ins on every section do none of these.

- **Reveals**: fade + 12–24px rise, 400–700ms, stagger 60–100ms, trigger once at ~20% visibility. Never hide the LCP/hero behind `opacity: 0` waiting for JS.
- **Micro-interactions**: 150–300ms on hover/press — card lifts, underline draws, magnetic buttons, number counters. Missing hover states read as machine-made.
- **Scroll effects**: one per page maximum (parallax ≤10–20% displacement, scroll-linked progress, pinned sequence). Transform/opacity only, never layout properties.
- **Libraries**: Motion v12 (`npm i motion`, import from `motion/react`) for React reveals/springs; GSAP 3.15+ with ScrollTrigger/SplitText (fully free since v3.13) for complex timelines; Lenis 1.3 for smooth scroll only when the direction demands it. CSS scroll-driven animations (`animation-timeline: view()`) work in Chrome/Edge 115+ and current Safari — guard with `@supports` and keep a no-animation fallback.
- **Always** honor `prefers-reduced-motion`: gate parallax/marquee/large translations behind the media query, keep opacity-only fades (Motion: `useReducedMotion`; GSAP: `gsap.matchMedia()`).

## Slop gates (run before delivering)

Fail any gate → fix before shipping:

1. No purple→blue gradient as hero/button/accent (unless the brief explicitly demands purple — then texture it).
2. No default-shadcn look: untouched `rounded-2xl shadow-lg p-6` cards, default zinc palette.
3. Not Inter/Poppins-only typography; a deliberate display face exists.
4. No emoji as feature icons; one consistent icon set (prefer Phosphor, Solar, or Heroicons over default Lucide).
5. Not centered-everything: at least the hero or one major section is asymmetric/left-aligned.
6. Radius, padding, and shadow vary with hierarchy — not identical on every element.
7. No "cardocalypse": cards only where grouping demands them, never cards inside cards.
8. No dark-neon glow borders / animated gradient borders ("the v0 signature") unless direction 8 explicitly calls for restraint-broken terminal aesthetics.
9. Hover + focus states exist on every interactive element; focus ring matches the accent.
10. One signature element exists; everything else is quiet.
11. Real assets: actual product UI, real numbers, no abstract 3D blobs, no generic laptop stock photos.
12. Copy is specific and founder-voiced ("Would the CEO actually say this?") — no "Build the future of work".
13. Sections vary in rhythm and density — not N identical icon+title+two-lines grids.
14. `::selection`, favicon, OG image, and `prefers-reduced-motion` handled.
15. Squint test: blurred, the page still shows a clear hierarchy — headline and CTA dominate.

## Tuning dials

When the user gives taste feedback, adjust these three dials (1–10) instead of guessing: **variance** (how far from safe defaults), **motion intensity**, **visual density**. "Make it bolder" = variance +2; "calmer" = motion −2, variance −1. State current dial values in the brief.

## Deliverable

Build mode: the written brief + `design/design-system.md` + implemented styles. Audit mode: numbered punch list sorted by impact, each item citing the gate or rule it violates. Always explain each major choice in one sentence tied to the brand answer from intake — never "modern and clean".
