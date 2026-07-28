# Art Direction Catalog (2025–2026)

Ten named directions with concrete recipes. Commit to exactly ONE per page; steal at most one accent idea from a second. Status labels reflect 2026 trend data (Figma trend report, Awwwards/Lapa Ninja galleries, mid-2026 retrospectives).

Selection heuristics by industry/audience:

| You're building for… | First candidates |
|---|---|
| Dev tools, AI, infrastructure | 8 Technical Mono, 1 Minimal Premium Dark (with a twist) |
| Broad B2B SaaS, fintech, health | 2 Soft Light SaaS, 4 Editorial |
| Consumer, education, community | 9 Playful Illustrated, 5 Neo-Brutalism (soft variant) |
| Agency, portfolio, brand site | 4 Editorial, 5 Neo-Brutalism |
| Premium lifestyle, finance, hospitality | 10 Luxury Serif |
| Feature-dense product | 3 Bento (as a feature-section layout inside another direction) |

---

## 1. Minimal Premium Dark ("Linear/Vercel-style") — MATURE / SATURATED

- **Essence:** engineered calm; a black instrument panel with one confident accent.
- **Status:** still everywhere, but the generic version is now the #1 "AI-generated" tell. Use only with a distinctive twist: unusual accent hue (acid lime, warm amber — not violet/cyan), editorial type, or real product UI as the hero.
- **Palette:** bg `#08090a`–`#0a0a0a`; headings near-white; body text at 60–75% opacity; ONE accent (e.g. acid-lime `oklch(0.92 0.22 115)` — Linear's current move — or indigo `#5e6ad2`); borders `rgba(255,255,255,0.06–0.12)` at 1px.
- **Type:** neutral grotesk (Geist, Inter-class) with tight tracking (−0.01…−0.03em) on headings; mono eyebrows/labels.
- **Surfaces:** radius 8–12px; elevation via lighter surface tiers + 1px borders, no drop shadows; radial "pool of light" glow behind the hero; blueprint/dot-grid background patterns.
- **Motion:** slow staggered fade+rise (400–700ms), border-glow hovers, logo marquee.
- **Pitfalls:** neon glow borders, purple gradients, glassmorphism cards → instant v0-slop.

## 2. Soft Light SaaS — STEADY DEFAULT

- **Essence:** Stripe-style restraint; warmth and air; trust-first.
- **Status:** the safest conversion baseline; differentiate via type and illustration or it disappears into the crowd.
- **Palette:** off-white/warm bg (`#fafafa`, `#f7f5f2`), gray-900 text, pastel section tints, one saturated accent for CTAs.
- **Type:** distinctive display (Bricolage Grotesque, Fraunces) + clean body (Inter, Work Sans).
- **Surfaces:** radius 12–16px; soft multi-layer low-alpha shadows; generous whitespace; hairline dividers.
- **Motion:** gentle 300–500ms reveals, subtle card lifts.
- **Pitfalls:** becoming "any SaaS": add one editorial moment (oversized number, serif pull-quote) and tinted—not gray—neutrals.

## 3. Bento Boxed — DOMINANT, PEAKING

- **Essence:** Apple-style modular card mosaic; density with order.
- **Status:** "the layout standard of 2026" — which means differentiation now comes from what's *inside* the cells. Best used as the features-section layout inside another direction, not as the whole page's personality.
- **Recipe:** varied cell spans on a 6-col grid with one hero cell (e.g. 4×2); consistent 12–24px gaps; radius 16–24px; per-card micro-visual, stat, or mini-demo; works dark or light.
- **Motion:** per-card hover states (lift/tilt/inner parallax); staggered entrance.
- **Pitfalls:** "cardocalypse" — every section boxed, cards in cards, identical cells. Cells must differ in size and content type.

## 4. Editorial / Typographic — RISING STRONGLY

- **Essence:** the headline IS the design; magazine confidence.
- **Recipe:** display type at 8–15vw (50%+ of viewport), line-height 0.9–1.05, mixed serif + grotesk in one headline, text-as-layout (oversized numbers as section markers, rules and columns from print), minimal ornament, lots of white (or cream) space.
- **Type:** Instrument Serif + Geist, Fraunces + Inter, or a variable grotesk stretched across weights.
- **Motion:** kinetic/split-text reveals (SplitText/Motion stagger per word), underline draws; little else.
- **When:** agencies, brands, premium positioning, anything fleeing the generic-SaaS look.
- **Pitfalls:** readability — body stays 16–18px with 45–75ch measure; contrast between display scale and body scale must be extreme, not incremental.

## 5. Neo-Brutalism — RISING (anti-AI authenticity signal)

- **Essence:** raw, loud, hand-built honesty.
- **Recipe:** 2–4px solid `#000` borders; hard offset shadows `4px 4px 0 #000` (zero blur); radius 0; clashing high-saturation colors (2–3, e.g. `#ff5941`, `#ffde00`, `#3300ff`) on cream or white; visible grid; system/mono type or chunky grotesk.
- **Motion:** hover = card translates into its own shadow (`translate(4px,4px)` + shadow removal); marquee tickers; instant, snappy (100–200ms).
- **Soft variant:** keep borders, add whitespace and friendlier type — works for B2B with attitude.
- **When:** youth/consumer brands, portfolios, dev tools with personality. Fatiguing for enterprise trust pages.
- **Pitfalls:** unreadable color-on-color text; check 4.5:1 contrast anyway.

## 6. Aurora / Gradient-Mesh — EVOLVED, CONDITIONAL

- **Essence:** atmospheric depth; light as a material.
- **Status:** survives only in its 2026 form — textured, contained, unusually colored. Untextured purple→blue mesh = canonical slop.
- **Recipe:** layered blurred radial/conic gradients over a dark base, slowly animated (20–40s loops); ALWAYS add a grain/noise overlay (SVG turbulence or a tiling noise PNG at 3–6% opacity); unusual hues (deep greens, teal→amber, oxblood); confine to the hero — content sections go clean.
- **Type:** clean grotesk so the background can speak.
- **Motion:** the gradient itself + calm reveals; premium versions run a small WebGL/shader canvas.
- **Pitfalls:** text contrast over the mesh (add a scrim); performance (blur is expensive — prerender to an image where possible).

## 7. Glassmorphism / Depth — FADING, USE RESTRAINED

- **Essence:** frosted layers communicating hierarchy.
- **Status:** dated as a page-wide style. Acceptable only as an accent: sticky nav, modal, one floating panel.
- **Recipe:** `backdrop-filter: blur(10–20px)`; 60–80% translucent fill; 1px `rgba(255,255,255,0.15)` border; over an image or gradient that makes the blur legible.
- **Pitfalls:** text contrast over blur; using glass on flat backgrounds (invisible = pointless); combining with direction 1 → v0-slop.

## 8. Technical Mono / Retro-Terminal — RISING

- **Essence:** command-line credibility; the interface as an instrument.
- **Status:** driven by Vercel/Linear/Raycast/Resend/PostHog brand systems; predicted mainstream in 2026.
- **Recipe:** monospace-led type (JetBrains Mono, Geist Mono, IBM Plex Mono, Space Mono) including headlines; stark B/W or green/amber-on-black; boxy 1px borders; ASCII diagrams, `$ command` snippets, blinking cursor; tabular data as decoration; uppercase letter-spaced labels.
- **Motion:** typewriter effects (sparingly), scanline/CRT touches for retro plays, instant hovers.
- **When:** developer/technical audiences, AI products, CLIs, APIs.
- **Pitfalls:** long body text in mono is tiring — pair a readable sans for paragraphs; keep the terminal theatre in hero/labels/code blocks.

## 9. Playful Illustrated / Collage — RISING (anti-AI signal)

- **Essence:** "a human made this" — imperfection as trust.
- **Recipe:** custom illustration or sticker graphics; cutout/torn-paper photo treatments; hand-drawn arrows, circles, underlines, annotations; dopamine-saturated palette on warm paper background; rotated elements (−3…+3deg).
- **Type:** rounded or hand-ish display (e.g. a chunky rounded grotesk) + clean body.
- **Motion:** bouncy springs (Motion spring presets), wiggle on hover, sticker "peel" effects.
- **When:** consumer, education, community, indie products.
- **Pitfalls:** stock "corporate Memphis" flat people = the previous generation's slop; assets must look bespoke.

## 10. Luxury Serif — RISING

- **Essence:** quiet money; editorial stillness.
- **Recipe:** cream/ivory bg (`#f6f2ea`) + near-black ink + one metallic-ish accent (bronze `#9a7b4f`, deep green, oxblood); high-contrast display serif (Playfair Display, Prata, Cormorant) with letter-spaced small-caps eyebrows; hairline rules 0.5–1px; muted full-bleed photography; radius 0–4px.
- **Motion:** slow 600–900ms ease-out fades, subtle image scale-on-scroll (1.05→1), nothing bouncy.
- **When:** premium lifestyle, finance, hospitality, high-ticket services.
- **Pitfalls:** thin serifs below 20px break — switch to the body face; gold-on-white fails contrast (darken the metallic for text).

---

## Cross-direction rules

- The direction decides borders/radius/shadows/motion — never mix surface languages (soft shadows + hard brutalist offsets = incoherence).
- Every direction still obeys the slop gates, contrast minimums, and `prefers-reduced-motion` from SKILL.md.
- Reference galleries when the user wants examples: awwwards.com, godly.website, land-book.com, lapa.ninja, saaspo.com, mobbin.com.
