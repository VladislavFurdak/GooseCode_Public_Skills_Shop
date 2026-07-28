# Page Recipes by Goal

Section orders that follow the sales conversation: promise → belief → proof → mechanism → objections → risk reversal → ask. Block anatomy lives in the `landing-blocks` skill; this file decides WHICH blocks and in WHAT order for each page goal.

Universal rules first:

- Promise + proof + CTA inside screen one (57% of viewing time is above the fold — NN/g).
- Page length = f(ticket size, traffic temperature, offer complexity). Warm traffic + cheap/simple offer → short page; cold traffic + expensive/novel offer → long page. When in doubt, build long and let scroll analytics argue for cuts.
- Every recipe ends with a full final-CTA section and a footer; every recipe repeats the CTA after major proof blocks.
- Median conversion benchmarks (Unbounce, for honest expectations): overall 6.6%, SaaS 3.8%, ecommerce ~4.2%, finance 8.4%, events 12.3%. Top-10% pages do 3–5× median.

## 1. SaaS — free trial / signup (simple, self-serve product)

Primary CTA: "Start free trial" (+ microcopy: no card / 14 days / cancel anytime). Secondary: "Watch demo" (ghost).

1. Navbar (anchors + CTA)
2. Hero — split or product-first; real UI screenshot
3. Logo cloud / ratings strip
4. How it works (3 steps — kills "sounds complicated")
5. Features — bento or zigzag, 3–6, each answering an objection · CTA repeat
6. Product showcase (big screenshot / 60s demo)
7. Testimonials (wall, quantified outcomes)
8. Stats band
9. Pricing (toggle, middle tier highlighted) · CTA repeat
10. FAQ (security, migration, cancellation)
11. Final CTA (restated promise + strongest quote)
12. Footer

No-card trials get 3–5× more signups (converting to paid at ~9% vs ~31% for card-gated — decide with the user which economics they want, and say which they chose).

## 2. SaaS — demo-first (high ACV, complex product)

Primary CTA: "Book a demo". Secondary: "See it in action" (2-min video).

1. Navbar
2. Hero — outcome headline for the buyer role, product visual
3. Logo cloud (enterprise logos matter most here)
4. Problem/agitation (name the expensive status quo)
5. Features by ROLE or use-case (tabs work well)
6. Comparison — "the old way vs with {Product}"
7. Case-study spotlight (one customer, numbers) · CTA repeat
8. Security/compliance badges (SOC 2, GDPR) — B2B objection killer
9. FAQ (procurement questions: SSO, data residency, contracts)
10. Final CTA — "Book a 20-min demo" + calendar-feel friction reducer ("pick a slot, no sales pressure")
11. Footer

Demo form: ≤4 fields (name, work email, company size) or multi-step; every extra field must earn its qualification value.

## 3. Waitlist / pre-launch

Primary CTA: email field, "Join the waitlist". The entire page is one screen + optional proof below.

1. (No navbar, or logo only)
2. Hero — form-first: outcome headline ("The inbox that answers itself — coming March"), single email field + button, "Join 2,314 others"
3. 3 teaser bullets or one product teaser visual (blurred/partial builds intrigue only if something real is shown)
4. Founder note — why we're building this (credibility substitute for missing proof)
5. Final CTA repeat (same form)
6. Minimal footer

Mechanics that decide success: single field only; social count beside the form; **referral loop on the confirmation state** ("share your link to jump the queue" — the Robinhood pre-launch mechanic, ~1M signups); zero competing links. ~20% conversion is achievable on warm launch traffic.

## 4. Lead-gen (downloadable / consultation / quote)

Primary CTA: the form itself. Traffic is usually cold paid → strip navigation entirely (attention ratio 1:1; nav removal tests: +40–100%).

1. Logo-only header
2. Hero — split: left = what they get + benefit bullets ("In this guide: …"), right = form card
3. Proof strip (ratings, "downloaded by 12,000 marketers")
4. What's inside / what happens next (3 bullets or steps)
5. Testimonial spotlight
6. FAQ (short — "is this a sales call?", privacy)
7. Final CTA repeat (same form or anchor to it)
8. Minimal footer + privacy link (required — you're collecting PII)

Form: 3–5 fields max (each cut ≈ measurable lift; 4→3 ≈ +50% in HubSpot data); labels above fields; privacy microcopy under the button ("No spam. Unsubscribe anytime."). Long qualification → multi-step, easy questions first, email last.

## 5. Mobile app

Primary CTA: store badges (device-detected: iOS → App Store, Android → Play; desktop → QR code beside badges).

1. Navbar (light)
2. Hero — phone frame with a REAL screen recording/screenshot; headline = the job the app does; badges + "★ 4.8 · 120k reviews"
3. Press/UGC strip
4. Feature carousel — one phone screen per feature, swipeable on mobile
5. Ratings & review quotes (store-style cards)
6. How it works (if onboarding is a selling point: "Set up in 30 seconds")
7. Final CTA — badges + QR again
8. Footer

Repeat store badges after every major section — the badge IS the CTA. Screenshots must match the current app version (mismatch = refund-grade distrust).

## 6. Ecommerce product (single-product landing)

Primary CTA: "Add to cart / Buy now" with price visible.

1. Navbar (minimal; cart icon)
2. Hero — gallery left (multiple angles, zoom), right: name, price, rating summary, variant pickers, buy button, shipping/returns/guarantee bullets directly under the button
3. Benefit blocks — ingredient/material/mechanism storytelling with photography
4. UGC / photo reviews strip
5. Comparison vs alternatives (or "why not the cheap one")
6. Reviews — rating distribution + filterable photo reviews (Baymard: vertical sections, not horizontal tabs — tabs make 27% of users miss content)
7. FAQ (sizing, shipping times, returns)
8. Final CTA — buy block repeat
9. Footer

Sticky add-to-cart bar on mobile scroll (+5–10% in published tests). Honest urgency only ("3 left" must be true). Product schema JSON-LD with offers + aggregateRating.

## 7. Event / webinar registration

Primary CTA: "Save my seat" (register form).

1. Logo header
2. Hero — title formula: benefit + audience + time commitment + credibility ("How SaaS CFOs close the books 4 days faster — live, 45 min, with the CFO of X"); date/time WITH timezone; form or CTA→form
3. Speakers — faces, names, titles (faces sell events)
4. Agenda — 3–5 bullets, each an outcome ("You'll leave with…")
5. Social proof — prior-session quotes/photos, registrant count
6. FAQ (recording? cost? who is it for?)
7. Final CTA + urgency (real seat cap or date proximity)
8. Minimal footer

Form: 2–4 fields. Registration pages convert 25–45%; expect ~half of registrants to attend — the confirmation page/email (add-to-calendar buttons) is part of the page's job.

## Localized pages (RU + EN, etc.)

Use App Router i18n-lite: `app/(ru)/page.tsx` + `app/en/page.tsx` (or `[locale]` segment) sharing section components; all strings come from `content/copy.{ru,en}.ts`; `alternates.languages` (`hreflang`) in metadata; localize the OG image too. Translate meaning, not words — CTAs and idioms are re-written per locale, and Cyrillic needs the font subset enabled.
