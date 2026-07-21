# CareerForge — Implementation Guide

This package gives you a **working front-end design** (`index.html`) plus **legal page copy** (`legal-policies.md`) for the digital career-services site you specified. Below is the exact, step-by-step path from these files to the full production stack (Next.js + Prisma + Stripe + Vercel) described in your brief.

## What's in this delivery

| File | What it is |
|---|---|
| `index.html` | Complete, responsive, accessible homepage — hero, services grid, FAQ accordion, contact form, checkout modal (front-end only), cookie banner. Open it directly in a browser to preview. |
| `legal-policies.md` | Draft copy for Privacy Policy, Terms & Conditions, Refund Policy, Cookie Policy, Accessibility Statement, and Disclaimer. **Needs solicitor review before publishing.** |
| `README.md` | This guide. |

`index.html` is a static preview of the real product — it demonstrates the design, copy, and interactions, but the checkout form, contact form, and cookie banner don't yet talk to a server. Steps 1–9 below turn it into the real, production system from your brief (Next.js, PostgreSQL, Stripe, file uploads, dashboards, GDPR tooling).

---

## Step 1 — Scaffold the Next.js project

```bash
npx create-next-app@latest careerforge --typescript --tailwind --app --eslint
cd careerforge
```

**Where files go:**
- Put the visual design from `index.html` into `app/page.tsx` as React components — split it into `components/Hero.tsx`, `components/Services.tsx`, `components/Faq.tsx`, `components/Contact.tsx`, `components/Footer.tsx`. Copy the CSS custom properties from the `<style>` block into `app/globals.css` under `:root`.
- Each legal page from `legal-policies.md` becomes its own route: `app/privacy-policy/page.tsx`, `app/terms-and-conditions/page.tsx`, `app/refund-policy/page.tsx`, `app/cookie-policy/page.tsx`, `app/accessibility-statement/page.tsx`, `app/disclaimer/page.tsx`.

## Step 2 — Install core dependencies

```bash
npm install prisma @prisma/client stripe @stripe/stripe-js resend zod react-hook-form @hookform/resolvers
npm install -D prisma
npx prisma init
```

## Step 3 — Database schema (PostgreSQL + Prisma)

Create `prisma/schema.prisma` with models for: `User`, `Order` (service, price, status, fileUrls[], notes, consentGivenAt), `Service` (title, price, deliveryWindow, description), `ContactMessage`, `ConsentLog` (for PECR/GDPR consent proof), and `CookiePreference`.

```bash
npx prisma migrate dev --name init
```

Host the database on [Supabase](https://supabase.com), [Neon](https://neon.tech), or [Railway](https://railway.app) and put the connection string in `.env` as `DATABASE_URL`.

## Step 4 — Authentication

Add [Auth.js](https://authjs.dev) (or Clerk) for the customer dashboard and a separate protected `/admin` route group. Store session data server-side; never trust client-only role checks.

```bash
npm install next-auth
```

## Step 5 — File uploads

Use Supabase Storage or AWS S3. On the API route that handles uploads:
1. Validate MIME type against an allow-list (`pdf`, `doc`, `docx`, `png`, `jpeg`) — reject everything else, especially executables.
2. Enforce a 20MB size limit server-side (not just in the UI).
3. Run an antivirus scan (e.g. ClamAV via a Lambda/worker, or a third-party scanning API) before the file is marked "available" on the order.
4. Store only the resulting object URL/key in the database, not the file itself.

## Step 6 — Stripe integration

1. Create Products/Prices in the Stripe Dashboard matching the six services and their GBP prices.
2. Add an API route `app/api/checkout/route.ts` that creates a Stripe Checkout Session (enable card, Apple Pay, Google Pay in the Stripe Dashboard's payment methods settings) and redirects the browser there — this replaces the `alert()` placeholder in the checkout modal's submit handler in `index.html`.
3. Add a webhook route `app/api/webhooks/stripe/route.ts` to listen for `checkout.session.completed`, mark the order as paid, and trigger the confirmation emails in Step 7.
4. Env vars: `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_WEBHOOK_SECRET`.

## Step 7 — Transactional email

Use [Resend](https://resend.com). Build templates for: order confirmation, payment confirmation, delivery confirmation, revision-request confirmation (to customer) and new-order/new-contact/new-upload alerts (to admin). Trigger these from the Stripe webhook (Step 6) and the contact-form API route.

## Step 8 — GDPR/PECR mechanics

- Wire the cookie banner in `index.html` (`cookieAcceptAll` / `cookieEssentialOnly`) to write a `CookiePreference` row server-side, not just `localStorage`, so consent is provable.
- Only load Google Analytics 4 / Microsoft Clarity scripts after analytics consent is given.
- Log a timestamped consent record (`ConsentLog`) whenever a checkbox is ticked on the checkout or contact form.
- Build a "Download my data" and "Delete my account" action into the customer dashboard, backed by API routes that export/delete the relevant database rows and files.

## Step 9 — Dashboards

- **Admin** (`app/admin/...`, protected route): orders table with status, customer details, uploaded files, a "mark complete" action, and a simple revenue total.
- **Customer** (`app/dashboard/...`, protected route): order status, upload additional files, download completed work, message thread, invoice download.

## Step 10 — SEO, performance, accessibility pass

- Move the `<meta>`, Open Graph, and JSON-LD block from `index.html` `<head>` into Next.js `generateMetadata()` per route.
- Add `app/sitemap.ts` and `app/robots.ts` (Next.js file conventions generate `sitemap.xml`/`robots.txt` automatically).
- Run Lighthouse and `axe DevTools` against the deployed preview; fix anything below AA contrast or the 95+ performance target — mainly image compression (use `next/image`) and deferring non-critical JS.

## Step 11 — Deploy

1. Push the repo to GitHub.
2. Import it into [Vercel](https://vercel.com/new).
3. Add all environment variables from the table below in the Vercel project settings.
4. Point your domain's DNS at Vercel and enable HTTPS (automatic on Vercel).

### Environment variables reference

```
DATABASE_URL=
NEXTAUTH_SECRET=
NEXTAUTH_URL=
STRIPE_SECRET_KEY=
STRIPE_PUBLISHABLE_KEY=
STRIPE_WEBHOOK_SECRET=
RESEND_API_KEY=
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
NEXT_PUBLIC_GA4_ID=
NEXT_PUBLIC_CLARITY_ID=
```

## After launch — future features to slot in later

The structure above (typed API routes, Prisma models, a component-based front end) is built so these can be added without a rewrite: AI CV review, AI cover letter generator, appointment booking, live chat, blog/knowledge base, discount codes, referral/affiliate system, newsletter, CRM integration, and a fuller analytics dashboard.

---

**Before publishing:** have a UK solicitor review `legal-policies.md`, and have a qualified accessibility auditor confirm the WCAG 2.2 AA claim once Step 10 is complete.
