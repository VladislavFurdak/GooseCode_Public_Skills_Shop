# Code Patterns for Landing Blocks

Next.js 16 App Router + Tailwind v4 + Motion v12 (`npm i motion`, import from `motion/react`). Sections are Server Components by default; only files marked `"use client"` ship JS. Adapt all colors/radii/type to the page's `design/design-system.md` — these skeletons are structure, not style.

## Section wrapper (rhythm)

```tsx
// components/sections/section.tsx (server)
export function Section({
  id, tint, children,
}: { id?: string; tint?: boolean; children: React.ReactNode }) {
  return (
    <section id={id} className={tint ? "bg-surface" : undefined}>
      <div className="mx-auto max-w-6xl px-4 py-16 md:px-6 md:py-24">{children}</div>
    </section>
  );
}
```

Alternate `tint` between sections for scroll texture. Prose blocks inside get `max-w-2xl`.

## Hero — split variant (LCP-safe)

```tsx
// components/sections/hero.tsx (server)
import Image from "next/image";
import shot from "@/public/product.png";

export function Hero() {
  return (
    <Section>
      <div className="grid items-center gap-12 lg:grid-cols-[45%_1fr]">
        <div>
          <p className="font-mono text-sm tracking-widest text-muted uppercase">For indie founders</p>
          <h1 className="mt-4 text-[clamp(2.5rem,1rem+5vw,4.5rem)] leading-[1.02] tracking-tight font-display">
            Invoices that chase themselves. <span className="text-accent">No spreadsheets.</span>
          </h1>
          <p className="mt-5 max-w-md text-lg text-muted">
            Acme watches your Stripe account and nudges late clients automatically.
          </p>
          <div className="mt-8 flex flex-wrap items-center gap-4">
            <a href="#pricing" className="inline-flex h-12 items-center rounded-lg bg-accent px-6 font-medium text-accent-foreground hover:opacity-90">
              Start my free trial
            </a>
            <a href="#demo" className="inline-flex h-12 items-center px-2 text-muted underline-offset-4 hover:underline">
              Watch 90s demo
            </a>
          </div>
          <p className="mt-3 text-sm text-muted">Free 14 days · No credit card · Cancel anytime</p>
        </div>
        <div className="rounded-xl border border-border bg-surface p-2 shadow-sm">
          <Image src={shot} alt="Acme dashboard showing overdue invoices" priority sizes="(min-width:1024px) 55vw, 100vw" className="rounded-lg" />
        </div>
      </div>
    </Section>
  );
}
```

Rules baked in: `priority` on the LCP image (never lazy, never hidden behind `opacity:0`), one `<h1>`, first-person CTA + microcopy, left-aligned split.

## Reveal on scroll (client island, reduced-motion safe)

```tsx
// components/reveal.tsx
"use client";
import { motion, useReducedMotion } from "motion/react";

export function Reveal({ children, delay = 0 }: { children: React.ReactNode; delay?: number }) {
  const reduce = useReducedMotion();
  return (
    <motion.div
      initial={reduce ? { opacity: 0 } : { opacity: 0, y: 20 }}
      whileInView={reduce ? { opacity: 1 } : { opacity: 1, y: 0 }}
      viewport={{ once: true, amount: 0.2 }}
      transition={{ duration: 0.55, delay, ease: [0.21, 0.5, 0.3, 1] }}
    >
      {children}
    </motion.div>
  );
}
```

Wrap below-the-fold section *content*, never the hero (LCP). Stagger siblings with `delay={i * 0.08}`. Pure-CSS alternative for zero JS: `animation-timeline: view()` behind `@supports`, visible-by-default fallback.

## Logo marquee (zero-JS CSS loop)

```tsx
// components/sections/logos.tsx (server)
const logos = ["/logos/a.svg", "/logos/b.svg" /* … */];

export function Logos() {
  return (
    <div className="py-10">
      <p className="text-center text-sm text-muted">Trusted by teams at</p>
      <div className="group mt-6 overflow-hidden [mask-image:linear-gradient(to_right,transparent,black_10%,black_90%,transparent)] motion-reduce:[mask-image:none]">
        <div className="flex w-max animate-marquee gap-16 group-hover:[animation-play-state:paused] motion-reduce:animate-none motion-reduce:w-full motion-reduce:flex-wrap motion-reduce:justify-center">
          {[...logos, ...logos].map((src, i) => (
            <img key={i} src={src} alt="" className="h-7 opacity-60 grayscale" aria-hidden={i >= logos.length} />
          ))}
        </div>
      </div>
    </div>
  );
}
```

```css
/* globals.css — Tailwind v4 */
@theme {
  --animate-marquee: marquee 32s linear infinite;
  @keyframes marquee { to { transform: translateX(-50%); } }
}
```

Content duplicated once for a seamless loop; hover pauses; `motion-reduce:` variants make it a static wrapped row.

## Bento features

```tsx
// components/sections/features.tsx (server)
const cells = [
  { span: "lg:col-span-4 lg:row-span-2", title: "Auto-reminders", body: "…", visual: <BigDemo /> },
  { span: "lg:col-span-2", title: "Stripe sync", body: "…", visual: <Stat n="2 min" label="setup" /> },
  { span: "lg:col-span-2", title: "Client portal", body: "…", visual: <MiniShot /> },
  // 4–6 cells, exactly one hero cell
];

export function Features() {
  return (
    <div className="grid gap-4 lg:grid-cols-6">
      {cells.map((c) => (
        <div key={c.title} className={`rounded-2xl border border-border bg-surface p-6 transition hover:border-accent/40 ${c.span}`}>
          {c.visual}
          <h3 className="mt-4 font-medium">{c.title}</h3>
          <p className="mt-1 text-sm text-muted">{c.body}</p>
        </div>
      ))}
    </div>
  );
}
```

Vary cell sizes AND content types (demo / stat / screenshot / list). If every cell is icon+title+text, use a plain grid instead — it's honester.

## Pricing with billing toggle (client island)

```tsx
// components/sections/pricing.tsx
"use client";
import { useState } from "react";

export function Pricing() {
  const [annual, setAnnual] = useState(true);
  const tiers = [
    { name: "Solo", m: 12, y: 9, features: ["1 workspace", "100 invoices/mo"] },
    { name: "Team", m: 29, y: 24, features: ["Unlimited workspaces", "Priority support"], popular: true },
  ];
  return (
    <div>
      <div className="flex items-center justify-center gap-3 text-sm">
        <span>Monthly</span>
        <button role="switch" aria-checked={annual} onClick={() => setAnnual(!annual)}
          className="h-6 w-11 rounded-full bg-border p-0.5 transition aria-checked:bg-accent">
          <span className={`block size-5 rounded-full bg-white transition ${annual ? "translate-x-5" : ""}`} />
        </button>
        <span>Annual <span className="text-accent">−20%</span></span>
      </div>
      <div className="mt-10 grid gap-6 md:grid-cols-2">
        {tiers.map((t) => (
          <div key={t.name} className={`rounded-2xl border p-8 ${t.popular ? "border-accent shadow-lg" : "border-border"}`}>
            {t.popular && <p className="text-xs font-medium text-accent">Most popular</p>}
            <h3 className="mt-1 text-lg font-medium">{t.name}</h3>
            <p className="mt-3 text-4xl font-display">${annual ? t.y : t.m}<span className="text-base text-muted">/mo</span></p>
            <ul className="mt-6 space-y-2 text-sm text-muted">{t.features.map((f) => <li key={f}>✓ {f}</li>)}</ul>
            <a href="/signup" className="mt-8 inline-flex h-11 w-full items-center justify-center rounded-lg bg-accent font-medium text-accent-foreground">
              Start free trial
            </a>
            <p className="mt-2 text-center text-xs text-muted">Cancel anytime</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

## FAQ — accordion + JSON-LD (server, zero JS)

```tsx
// components/sections/faq.tsx (server)
const faqs = [
  { q: "Can I cancel anytime?", a: "Yes — one click in settings, no emails, no calls." },
  { q: "Do you store my card details?", a: "No. Payments run through Stripe; we never see card numbers." },
];

export function Faq() {
  const jsonLd = {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    mainEntity: faqs.map((f) => ({
      "@type": "Question", name: f.q,
      acceptedAnswer: { "@type": "Answer", text: f.a },
    })),
  };
  return (
    <div className="mx-auto max-w-2xl">
      <script type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd).replace(/</g, "\\u003c") }} />
      <div className="divide-y divide-border border-y border-border">
        {faqs.map((f) => (
          <details key={f.q} className="group py-4">
            <summary className="flex cursor-pointer list-none items-center justify-between font-medium">
              {f.q}<span className="transition group-open:rotate-45">+</span>
            </summary>
            <p className="mt-2 text-muted">{f.a}</p>
          </details>
        ))}
      </div>
    </div>
  );
}
```

Native `<details>` = accessible, indexable, zero JS. The `<` escaping in JSON-LD is the Next.js-documented XSS guard.

## Sticky mobile CTA (appears after hero)

```tsx
// components/sticky-cta.tsx
"use client";
import { useEffect, useRef, useState } from "react";

export function StickyCta() {
  const [show, setShow] = useState(false);
  const sentinel = useRef<HTMLDivElement>(null);
  useEffect(() => {
    const io = new IntersectionObserver(([e]) => setShow(!e.isIntersecting));
    if (sentinel.current) io.observe(sentinel.current);
    return () => io.disconnect();
  }, []);
  return (
    <>
      <div ref={sentinel} aria-hidden className="absolute top-[85vh]" />
      <div className={`fixed inset-x-0 bottom-0 z-40 border-t border-border bg-bg/90 p-3 backdrop-blur transition-transform md:hidden ${show ? "translate-y-0" : "translate-y-full"}`}>
        <a href="#pricing" className="flex h-12 items-center justify-center rounded-lg bg-accent font-medium text-accent-foreground">
          Start my free trial
        </a>
      </div>
    </>
  );
}
```

Render the sentinel right after the hero. Never covers content on desktop (`md:hidden`); measure conversions, not clicks.

## Stats count-up (client island)

```tsx
// components/stat-number.tsx
"use client";
import { animate, useInView, useReducedMotion } from "motion/react";
import { useEffect, useRef } from "react";

export function StatNumber({ value, suffix = "" }: { value: number; suffix?: string }) {
  const ref = useRef<HTMLSpanElement>(null);
  const inView = useInView(ref, { once: true });
  const reduce = useReducedMotion();
  useEffect(() => {
    if (!inView || !ref.current) return;
    if (reduce) { ref.current.textContent = value.toLocaleString() + suffix; return; }
    const controls = animate(0, value, {
      duration: 1.2, ease: "circOut",
      onUpdate: (v) => { ref.current!.textContent = Math.round(v).toLocaleString() + suffix; },
    });
    return () => controls.stop();
  }, [inView, reduce, value, suffix]);
  return <span ref={ref} className="font-display text-5xl tabular-nums">0{suffix}</span>;
}
```

## Testimonial card (server)

```tsx
export function Testimonial({ quote, name, role, img, metric }: TProps) {
  return (
    <figure className="rounded-2xl border border-border bg-surface p-6">
      {metric && <p className="font-display text-3xl text-accent">{metric}</p>}
      <blockquote className="mt-3 text-pretty">
        “{quote /* bold the key result phrase with <strong> in the data */}”
      </blockquote>
      <figcaption className="mt-4 flex items-center gap-3">
        <img src={img} alt="" className="size-10 rounded-full object-cover" />
        <div className="text-sm"><p className="font-medium">{name}</p><p className="text-muted">{role}</p></div>
      </figcaption>
    </figure>
  );
}
```

Real photos, real names, quantified outcomes — or don't ship the block.
