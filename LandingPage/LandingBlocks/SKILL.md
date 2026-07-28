---
name: landing-blocks
description: Catalog of proven landing page sections (blocks) with variants, conversion rules, and Next.js/Tailwind build notes — hero, logo cloud, features/bento, how-it-works, testimonials, stats, comparison, pricing, FAQ, final CTA, footer. Use when composing or reordering a landing page, when the user asks to add or improve any specific page section (a hero, a pricing block, a testimonials section, an FAQ), or wants ready component libraries for marketing blocks.
---

# Landing Blocks — Section Catalog

Sections of a landing page are moves in one sales conversation: promise → belief → proof → mechanism → objections → risk reversal → ask. This file tells you which blocks exist, when each earns its place, and how to build it. Page-level ordering by goal (SaaS / waitlist / lead-gen / app / ecommerce / event) lives in the `landing-page-builder` skill; copy formulas live in `landing-cta`; visual language in `landing-design`.

Three laws that govern every block:

1. **Attention ratio → 1:1.** One goal per page, one place to click. Strip nav links on paid-traffic pages; every block either advances the one conversion or gets cut.
2. **57% of viewing time is above the fold, 74% in the first two screens** (NN/g). Blocks are ordered by descending objection frequency — the deeper the block, the more skeptical its reader.
3. **Every repetition of the CTA is paired with a new reason** (fresh proof or restated benefit), never a naked button.

Code skeletons for the marked blocks: [references/code-patterns.md](references/code-patterns.md).

## Quick index

| # | Block | Job | Skip when |
|---|-------|-----|-----------|
| 0 | Announcement bar | urgency/news in one line | nothing timely to say |
| 1 | Navbar | orientation + CTA | paid-traffic page (strip to logo+CTA) |
| 2 | Hero | promise + proof + ask in one screen | never skip |
| 3 | Logo cloud / proof strip | credibility before scroll investment | no logos/numbers yet (use ratings/counts) |
| 4 | Problem / agitation | make the pain vivid | product-aware traffic, obvious pain |
| 5 | How it works | kill "sounds complicated" | one-step products |
| 6 | Features / bento | benefits + objection handling | — |
| 7 | Product showcase | "show me the actual thing" | pre-product waitlists |
| 8 | Testimonials | borrowed trust | none exist (never fake them) |
| 9 | Stats band | concrete scale/outcome numbers | numbers unimpressive |
| 10 | Comparison | frame the alternative | no meaningful rival ("old way" counts) |
| 11 | Pricing | pre-qualify, kill price anxiety | enterprise/custom deals |
| 12 | FAQ | catch stray objections + SEO schema | — |
| 13 | Final CTA | restate promise, last ask | never skip |
| 14 | Footer | legitimacy, legal, secondary paths | never skip |
| + | Waitlist form, integrations grid, security badges, founder note | situational (see end) | |

---

## 0. Announcement bar

One line above the nav: launch, funding, discount, event. Dismissible, ~40px, accent background or inverted colors. Link the whole bar, not a tiny "→". Reserve its height to avoid CLS. Skip it if there's nothing genuinely timely — a permanent "🎉 We launched!" is noise.

## 1. Navbar

Logo left; 3–5 anchor links center/left (Features, Pricing, FAQ); primary CTA button right (same label as hero CTA). Mobile: logo + CTA + hamburger. Sticky with a background that appears on scroll (transparent → surface + border). On dedicated ad landing pages remove all links except the logo (attention ratio: removing nav has produced +40–100% lifts in published tests).

## 2. Hero ⭐ (code pattern available)

The one block that must work alone: 5 elements — headline (value), subheadline (mechanism + audience), primary CTA + risk-reversal microcopy, proof element, product visual. Half of total design effort goes here.

**Variants — choose by what sells the product best:**
- **Split** (text left ~45%, visual right) — default for products with a good screenshot. Left-align; avoid centered-everything.
- **Centered statement** — when the words are the product (bold claim + short subhead + CTA); pair with Editorial direction; visual below.
- **Product-first** — big product UI front and center, angled/framed, text above. For visually impressive apps.
- **Form-first** — email field IS the hero CTA (waitlists, lead-gen). Single field + button on one line.
- **Video/demo** — click-to-play with a real poster frame; never autoplay with sound; never a substitute for a headline.
- **Typographic** — 8–15vw display type, no visual (agencies, brands).

**Build:** `next/image` with `priority` on the hero visual (it's the LCP element — never lazy-load it, never start it at `opacity:0`); `<h1>` exactly once; buttons ≥44px tall on mobile; screenshot in a browser/device frame with subtle border, not floating raw.

## 3. Logo cloud / social proof strip ⭐

5–8 recognizable customer/press logos, monochromed to the muted text color, or "★ 4.8 on G2 · 18,000+ reviews" when logos are weak. Sits immediately under the hero — credibility before the visitor invests scroll time. More than ~8 logos → marquee loop (CSS transform keyframes, duplicated list, gradient-mask edges, pause on hover, static under `prefers-reduced-motion`). Label it ("Trusted by teams at…") — unlabeled logo soup reads as decoration.

## 4. Problem / agitation

Name the pain in the customer's own words before selling the fix (PAS). Formats: 3 pain cards ("Sound familiar?"), a short narrative paragraph, or a before/after split. Concrete beats abstract: "4 hours every Friday copying numbers between tabs", not "inefficient workflows". Critical for problem-aware cold traffic and new-category products; skip for retargeting/most-aware pages where it delays the offer.

## 5. How it works

3 (max 4) numbered steps, each: verb-first title + one sentence + small visual/icon. Horizontal on desktop, vertical on mobile. The job is labor-and-confusion reduction — "Setup in 2 minutes" energy. Numbers are load-bearing: they promise the list is short. End the block with a CTA repeat when the flow feels complete.

## 6. Features / benefits ⭐

Each feature written as an outcome: value-prop header (not "Empower your workflow") + 1–2 sentence objection-handler + real product visual. 3–6 features that carry a running narrative back to the hero promise.

**Layouts:**
- **Bento grid** — 4–8 parallel capabilities; 6-col grid, varied spans, one hero cell, per-cell visual/stat/mini-demo; cells must differ in size AND content type (see design skill re: cardocalypse).
- **Alternating zigzag** — 2–4 deep features; image/text alternating sides; room for real screenshots per feature.
- **3-col grid** — quick scannable parity features; vary it from the default icon+title+two-lines pattern or it reads as template filler.
- **Tabs / interactive showcase** — many features, one screenshot area; tab switch swaps the visual. Client island.

## 7. Product showcase

One big moment of "here's the actual thing": full-width screenshot in a frame, 30–60s product GIF/video, or an interactive embed. Place after features (or merged with hero for product-first pages). Real UI only — blurred or fake UI destroys the trust the block exists to build. Lazy-load below the fold; `next/dynamic` for heavy embeds.

## 8. Testimonials ⭐

Strong testimonial = specific quantified outcome + full attribution (name, title, company, photo) + it answers a named objection + key phrase bolded. Photos measurably beat logos for recall (CXL). Map each quote to the objection of the adjacent section; save the strongest for beside the final CTA.

**Layouts:** 3-col masonry wall (6–9 quotes, varied lengths); single spotlight (one hero quote + big portrait + metric); carousel only when auto-advance is off and arrows are visible; tweet/Slack-screenshot style for dev/consumer authenticity (real screenshots, real handles).

## 9. Stats band

3–4 numbers with labels: users, hours saved, uptime, NPS, GMV. Precise beats round ("18,432 teams" > "thousands"). Big mono or display numerals; optional count-up animation on first view (respect reduced motion). Skip if numbers are small — "127 users" undermines more than it proves; use a different proof type instead.

## 10. Comparison ("us vs them")

Lead with one consistent argument, support it, restate it — never a neutral feature matrix. Formats: 2-col "The old way / With {Product}" (safe, no competitor named) or feature table vs 1–2 named competitors (check marks favor you, but every row must be true). Place after features/testimonials, before pricing — it's a late-stage objection block for solution/product-aware readers.

## 11. Pricing ⭐

Show pricing if at all possible — transparency pre-qualifies and shortens the sales journey; hide it only when deal size genuinely varies wildly. 2–3 tiers, middle tier highlighted ("Most popular"), annual/monthly toggle with the discount stated, per-tier CTA (same primary action), feature list front-loaded with differences (not 20 identical rows), risk reversal under buttons ("Cancel anytime"). Single-price products: one big card + what's included. Enterprise: "Talk to us" tier without fake "$Custom" theatrics.

## 12. FAQ ⭐

5–8 questions = the objections that had no natural home above: security, migration, cancellation, refunds, "how is this different from X", data ownership. Answers 2–4 sentences, honest, link deeper docs. Accordion (native `<details>` styled, or one-open-at-a-time client component). Add `FAQPage` JSON-LD (server-rendered; note: Google shows FAQ rich results mainly for authoritative sites since 2023, but the schema still feeds AI answer engines). A question that's really an objection belongs in copy up the page too — FAQ is the catch-all, not the hiding place.

## 13. Final CTA ⭐

A full section, never a lone button: restate the core promise (fresh phrasing, not a copy-paste of the hero), primary CTA + risk-reversal microcopy, strongest testimonial or proof number beside it. High-contrast treatment — inverted colors or accent background. **Founder's note variant** (strong for indie/early products): 3–5 sentence letter — their problem → you own it → the happy ending → signature + photo.

## 14. Footer

Legitimacy signal: logo + one-line description, link columns (Product / Company / Legal), contact, socials, privacy/terms (required if you collect emails), © year. On stripped ad pages keep a minimal footer — zero footer reads as scam.

## Situational blocks

- **Waitlist form block** ⭐ — single email field + outcome headline + "join {n} others" + referral loop on the confirmation ("share to jump the queue" — the Robinhood mechanic). Server Action implementation in `landing-page-builder` references.
- **Integrations grid** — logos of tools you connect to; label with the count ("Works with 40+ tools"). It's a feature block, not social proof.
- **Security/compliance badges** — SOC 2, GDPR, ISO logos near pricing/final CTA for B2B; link real reports.
- **Newsletter block** — only on content-heavy sites; on a landing it competes with the primary CTA (attention ratio) — usually cut.

## Assembly rules

- Alternate section background rhythm (bg → surface tint → bg) so scrolling has texture; don't box every section.
- Vertical padding rhythm: consistent scale (e.g. `py-16 md:py-24`), hero taller; wide sections `max-w-6xl`, prose `max-w-2xl`.
- CTA cadence: hero → after features or how-it-works → after testimonials/pricing → final. Every ~1.5–2 screens on long pages.
- Mobile-first check: 83% of visits are mobile and convert worse — audit the phone rendering of every block first, sticky bottom CTA on long mobile pages (appears after hero CTA scrolls away ⭐).
- Component libraries worth pulling from (verify license per component): **Tailark** (MIT, shadcn marketing blocks — fastest full-landing assembly), **Magic UI** (MIT — Marquee, Bento, animated text), **Motion Primitives** (MIT — tasteful micro-interactions), **Origin UI** (MIT primitives), **HyperUI** (MIT plain Tailwind), **shadcnblocks** (large catalog, partly paid), **Aceternity UI** (flashy effects — use sparingly, they're becoming their own cliché). Restyle anything you pull to the page's design system — default-styled imports are the slop the design skill exists to prevent.

## Pre-delivery checklist

- [ ] Every block advances the single conversion goal; nothing decorative survives
- [ ] Hero passes the 5-element check; ~50% of effort spent there
- [ ] Proof strip within the first two screens
- [ ] Each CTA repetition paired with new proof/benefit
- [ ] Section layouts vary (no three identical grids in a row)
- [ ] Mobile rendering checked block-by-block; sticky CTA if page > 4 screens
- [ ] All imported components restyled to the design system
- [ ] FAQ carries JSON-LD; footer has legal links
