# Next.js + Vercel Reference for Landing Pages

Verified mid-2026. If versions have moved on, the patterns still hold — check changelogs before pinning.

| Piece | State (mid-2026) |
|---|---|
| Next.js | 16.x stable (16.2 line; 15.5 = maintenance LTS). Turbopack default. PPR is stable via `cacheComponents`. `middleware.ts` → renamed `proxy.ts`. `params`/`searchParams` are async. |
| Tailwind CSS | v4 (4.3): CSS-first config, no `tailwind.config.js`, OKLCH palette |
| Motion | v12, package `motion`, import `motion/react` (Framer Motion's successor) |
| GSAP | 3.15+, fully free incl. ScrollTrigger/SplitText (since 3.13, Webflow-owned) |
| Lenis | 1.3.x — smooth scroll, only when the art direction demands it |
| shadcn/ui | CLI 3.0, namespaced registries (`npx shadcn add @tailark/hero-1` style) |
| React | 19.x — Server Components, Server Actions, `useActionState` |

## Scaffold

```bash
npx create-next-app@latest my-landing --typescript --tailwind --app --turbopack --no-src-dir --import-alias "@/*"
cd my-landing
npm i motion
npm i @vercel/analytics @vercel/speed-insights
# optional: npx shadcn@latest init   (only if pulling registry blocks)
```

A landing is static: no data fetching in page components → Next prerenders everything at build. Add `export const dynamic = "error"` in `app/page.tsx` to *guarantee* nothing accidentally opts into dynamic rendering. CMS-driven content → ISR: `export const revalidate = 3600`.

`output: "export"` (pure static hosting) exists but loses Server Actions, the image optimization API, and ISR — on Vercel there's no reason to use it.

## Tailwind v4 tokens (wire design-system.md here)

`globals.css`:

```css
@import "tailwindcss";

@theme {
  /* from design/design-system.md — OKLCH, semantic names */
  --color-bg: oklch(0.985 0.002 90);
  --color-surface: oklch(0.955 0.004 90);
  --color-text: oklch(0.21 0.01 90);
  --color-muted: oklch(0.45 0.008 90);
  --color-border: oklch(0.9 0.004 90);
  --color-accent: oklch(0.62 0.19 25);
  --color-accent-foreground: oklch(0.99 0 0);

  --font-display: var(--font-bricolage);
  --font-sans: var(--font-inter);
  --font-mono: var(--font-jetbrains);

  --animate-marquee: marquee 32s linear infinite;
  @keyframes marquee { to { transform: translateX(-50%); } }
}
```

Usage: `bg-bg text-text border-border font-display bg-accent` etc. Every token doubles as a runtime CSS variable (`var(--color-accent)`) usable from Motion/GSAP.

v3 → v4 tripwires: `@tailwind base/components/utilities` directives are gone (just `@import "tailwindcss"`); `bg-gradient-*` → `bg-linear-*`; shadow/radius/blur scales shifted (`shadow-sm`→`shadow-xs`, `shadow`→`shadow-sm`); default border color is `currentColor`; needs Safari 16.4+/Chrome 111+.

## Fonts

```tsx
// app/layout.tsx
import { Bricolage_Grotesque, Inter, JetBrains_Mono } from "next/font/google";

const display = Bricolage_Grotesque({ subsets: ["latin"], variable: "--font-bricolage" });
const sans = Inter({ subsets: ["latin", "cyrillic"], variable: "--font-inter" });
const mono = JetBrains_Mono({ subsets: ["latin"], variable: "--font-jetbrains" });

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${display.variable} ${sans.variable} ${mono.variable}`}>
      <body className="bg-bg font-sans text-text antialiased">{children}</body>
    </html>
  );
}
```

`next/font` self-hosts at build (zero external requests), injects metric-adjusted fallbacks (~zero CLS), `display: swap` by default. Budget: ≤2 families + 1 mono. Cyrillic subset when the page has Russian.

## Metadata

```tsx
// app/layout.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  metadataBase: new URL("https://example.com"),
  title: { default: "Acme — invoices that chase themselves", template: "%s · Acme" },
  description: "Acme watches Stripe and nudges late clients automatically. Setup in 2 minutes.",
  alternates: { canonical: "/" },
  openGraph: {
    type: "website", siteName: "Acme", url: "/",
    title: "Acme — invoices that chase themselves",
    description: "Automatic reminders for late invoices.",
  },
  twitter: { card: "summary_large_image" },
};
```

Canonical is mandatory on pages receiving UTM traffic. Title ≤ 60 chars, description 140–160, both restating the hero promise (message match extends to the SERP).

## OG image (design it — it's part of the hero effort)

```tsx
// app/opengraph-image.tsx
import { ImageResponse } from "next/og";

export const size = { width: 1200, height: 630 };
export const contentType = "image/png";
export const alt = "Acme — invoices that chase themselves";

export default function OgImage() {
  return new ImageResponse(
    (
      <div style={{
        width: "100%", height: "100%", display: "flex", flexDirection: "column",
        justifyContent: "center", padding: 80, background: "#0a0a0a", color: "#fafafa",
      }}>
        <div style={{ fontSize: 28, color: "#9a9a9a" }}>acme.com</div>
        <div style={{ fontSize: 72, fontWeight: 700, lineHeight: 1.05, marginTop: 16 }}>
          Invoices that chase themselves
        </div>
      </div>
    ),
    size,
  );
}
```

Rendered at the edge, cached. Subset of CSS (flex only, no grid). Load a custom font via `fetch` + `fonts` option for brand type. Same file convention: `twitter-image.tsx` (usually re-export). The OG card is what 100% of social clicks see before the page — apply the design system's direction, not a generic gradient.

## sitemap.ts / robots.ts

```tsx
// app/sitemap.ts
import type { MetadataRoute } from "next";
export default function sitemap(): MetadataRoute.Sitemap {
  return [{ url: "https://example.com", lastModified: new Date(), priority: 1 }];
}

// app/robots.ts
import type { MetadataRoute } from "next";
export default function robots(): MetadataRoute.Robots {
  return { rules: { userAgent: "*", allow: "/" }, sitemap: "https://example.com/sitemap.xml" };
}
```

## JSON-LD

Server-render a `<script type="application/ld+json">`; escape `<` per Next docs (XSS guard):

```tsx
function JsonLd({ data }: { data: object }) {
  return <script type="application/ld+json"
    dangerouslySetInnerHTML={{ __html: JSON.stringify(data).replace(/</g, "\\u003c") }} />;
}
```

Ship on a landing: **Organization** (name, url, logo, sameAs) in the layout; **SoftwareApplication** (SaaS — with `offers`) or **Product** on the page; **FAQPage** with the FAQ block (Google limits FAQ rich results to authoritative sites since 2023, but the schema feeds AI answer engines — cheap to ship). Validate: validator.schema.org + Google Rich Results Test.

## Waitlist / contact form — Server Action

```tsx
// app/actions.ts
"use server";
import { z } from "zod";

const schema = z.object({ email: z.string().email(), company: z.string().max(0) }); // company = honeypot

export async function joinWaitlist(_prev: unknown, formData: FormData) {
  const parsed = schema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { ok: false, error: "Enter a valid email" };
  if (parsed.data.company) return { ok: true }; // bot filled the honeypot — pretend success
  // persist: DB / Resend audience / KV / Google Sheet — user's choice
  return { ok: true };
}
```

```tsx
// components/sections/waitlist-form.tsx
"use client";
import { useActionState } from "react";
import { joinWaitlist } from "@/app/actions";

export function WaitlistForm() {
  const [state, action, pending] = useActionState(joinWaitlist, null);
  if (state?.ok) return <p className="text-accent">You're on the list — check your inbox. Share your link to jump the queue →</p>;
  return (
    <form action={action} className="flex w-full max-w-md gap-2">
      <input name="company" tabIndex={-1} autoComplete="off" aria-hidden className="hidden" />
      <label className="sr-only" htmlFor="email">Email</label>
      <input id="email" name="email" type="email" required placeholder="you@work.com"
        className="h-12 flex-1 rounded-lg border border-border bg-surface px-4" />
      <button disabled={pending} className="h-12 rounded-lg bg-accent px-5 font-medium text-accent-foreground disabled:opacity-60">
        {pending ? "Joining…" : "Join waitlist"}
      </button>
      {state?.error && <p role="alert" className="text-sm text-red-600">{state.error}</p>}
    </form>
  );
}
```

Progressive enhancement (works without JS), pending state, honeypot instead of CAPTCHA (a CAPTCHA on a waitlist murders conversion). Errors preserve input; success message carries the referral hook.

## Analytics, events, A/B

```tsx
// app/layout.tsx (inside <body>)
import { Analytics } from "@vercel/analytics/react";
import { SpeedInsights } from "@vercel/speed-insights/next";
<Analytics /> <SpeedInsights />
```

- Web Analytics: cookieless; Hobby = 50k events/mo. Track CTA clicks/submits: `import { track } from "@vercel/analytics"` → `track("cta_click", { location: "hero" })`.
- Speed Insights: real-user CWV; Hobby = first 10k events.
- **A/B**: Flags SDK (`flags` package) — typed flags + the **precompute** pattern: `proxy.ts` evaluates flags and rewrites to a prerendered static variant → zero flicker, zero CLS, static-speed A/B. Backing store: Edge Config (sub-ms reads). Wire only when the user actually wants experiments; a page with no traffic needs copy iterations, not infra.

## Images

- Hero/LCP: `priority`, honest `sizes`, AVIF auto. Everything below the fold: default lazy.
- Screenshots: export at 2× display size, not 4000px originals.
- Vercel Hobby: 5,000 image transformations/mo — fine for a landing, but don't feed a 50-image gallery through the optimizer casually.

## Third-party scripts (the #1 Lighthouse killer)

Sync GTM/chat/pixel tags can double LCP. Rules: `@next/third-parties` for GA/GTM/YouTube (`<YouTubeEmbed>` = facade); everything else `next/script` with `strategy="afterInteractive"` (analytics) or `"lazyOnload"` (chat, support widgets); video embeds always behind a poster-image facade; cookie banner (if legally needed) async with reserved space.

## Performance checklist (Phase 7)

- [ ] CWV lab pass, mobile: LCP ≤ 2.5s, INP ≤ 200ms, CLS ≤ 0.1
- [ ] LCP element is the hero image/headline with `priority`, not hidden by animation
- [ ] Fonts: `next/font` only, ≤3 files, no FOUT jump
- [ ] Zero layout shift from bars/banners/embeds/marquees (reserved dimensions)
- [ ] Client islands audited: every `"use client"` justified; heavy ones behind `next/dynamic`
- [ ] Animations: transform/opacity only; `prefers-reduced-motion` verified in DevTools emulation
- [ ] Third-party scripts deferred/facaded; test with them ENABLED — that's what users get
- [ ] Keyboard: tab order reaches the primary CTA fast; visible focus rings; skip-link if nav is long
- [ ] `next build` clean; no `params` sync-access warnings (Next 16: they're async)

## Deploy

```bash
vercel          # first deploy → preview
vercel --prod   # promote
```

Or push to GitHub → import in Vercel → every branch gets a preview URL, `main` = production. Set env vars (`vercel env add`) for form persistence keys. Custom domain: Vercel dashboard → Domains (auto-HTTPS). **Hobby tier is non-commercial** — a commercial landing needs Pro ($20/user/mo). Analytics/Speed Insights toggles live in the project dashboard, plus enable "Deployment Protection" off for public previews you share with the client.
