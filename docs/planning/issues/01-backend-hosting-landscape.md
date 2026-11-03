# 01: Backend and hosting landscape (facts as of 2026-10-29)

Type: research
Status: resolved
Blocked by: none
Claimed by: research subagent, 2026-10-29

## Question

What do the candidate backends and hosts offer on their free tiers today, what are their hard limits and gotchas for
a low-traffic hobby app with the skillassert profile, what is the first paid step, and how portable is each?
Candidates: Firebase, Supabase, a cheap VPS (Hetzner and alternatives), static SPA hosts (Cloudflare Pages, Vercel,
Netlify, Firebase Hosting, GitHub Pages), edge or serverless API options (Cloudflare Workers + D1, Vercel and Netlify
functions), managed Postgres free tiers (Neon and peers), and the lighter BaaS options PocketBase and Appwrite.

## Why it is on the map

The backend choice (round 1, Q2 and Q3) is the highest-impact decision, and price and limit knowledge goes stale
fast. The findings feed round 2.

## Answer

Resolved 2026-10-29. Full report with sources: [research/01-backend-hosting-landscape.md](../research/01-backend-hosting-landscape.md).
Decision-relevant facts for this app:

- **Supabase Free** fits the relational model: Postgres 500 MB, 50K MAU, anonymous and Google sign-in, row-level
  security, built-in full-text search, EU regions, self-hostable via Docker Compose. Gotchas: a project pauses after
  a week of low DB activity (a few requests a day avoid it); default SMTP sends 2 emails/hour to team addresses only,
  so email sign-up needs custom SMTP from day one; Edge Functions allow 2 s CPU and 150 s wall clock, too small for
  an LLM batch; no backups on Free; Pro is $25/month plus VAT, above the EUR 20 budget, so the plan must stay on Free
  or self-host.
- **Firebase** is a poor fit: Firestore is a document store, text search needs the Enterprise edition, Cloud
  Functions require Blaze with a card and no hard spend cap, Spark caps at 3,000 daily active users. Its anonymous
  account linking keeps the user id, which is the one thing it does better.
- **A cheap EU VPS** for the NestJS content service: OVH VPS-1 2 vCPU/4 GB from EUR 3.81 excl. VAT; Hetzner
  CX23/CAX11 EUR 5.99 to 6.49 excl. VAT with IPv4 but flagged "not available" on the product pages; Netcup VPS 500
  EUR 9.50 incl. VAT. Oracle Always Free has reclaim and account-termination caveats.
- **SPA hosting**: Cloudflare Pages has unlimited static requests, 500 builds/month, custom domain with TLS, but an
  apex domain needs Cloudflare nameservers (the domain is on Porkbun nameservers today; moving NS is free). Vercel
  Hobby is non-commercial only and GitHub Pages bars commercial SaaS, which a future premium tier would violate.
  Netlify Free runs on 300 credits/month and pauses every site at zero.
- **Serverless for the LLM batch** does not fit the free tiers: Workers Free 10 ms CPU, Vercel Hobby one cron a day,
  Supabase Edge 150 s.
- **Ruled out**: PocketBase (repo archived 2026-10-26 and moved, "not recommended for production-critical apps", no
  anonymous sessions) and Appwrite Free (pauses after 7 days without console activity, deleted after 90 days).
- **Neon Free** is a fallback Postgres: 100 CU-hours/project/month, 1 GB, Frankfurt, scale to zero after 5 min.

Open facts carried into tickets: whether Supabase keeps the user id on anonymous-to-permanent conversion (docs do
not state it), Hetzner entry-plan availability, and a free transactional-email provider for Supabase SMTP.
