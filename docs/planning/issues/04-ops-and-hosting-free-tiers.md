# 04: Operations and hosting mechanics on free tiers (facts as of 2026-10-29)

Type: research
Status: resolved
Blocked by: none
Claimed by: research subagent, 2026-10-29

## Question

Facts needed to wire the operations baseline (Q31, Q32, Q33, Q34): free object storage for nightly database dumps
(Cloudflare R2, Backblaze B2), free uptime monitors, free error tracking (Sentry and a self-hosted alternative),
Cloudflare Web Analytics, Cloudflare Pages specifics for an Angular app with prerendered routes and SPA fallback,
GitHub Container Registry and GitHub Actions allowances for a public repository, and Caddy automatic TLS basics.

## Answer

Resolved 2026-10-29. Full report with sources: [research/04-ops-and-hosting-free-tiers.md](../research/04-ops-and-hosting-free-tiers.md).
Picks written into design.md section 13 (all my calls on verified facts):

- **Backups**: Backblaze B2, EU Central (Amsterdam), 10 GB free, no card at sign-up, S3-compatible; different vendor
  from the frontend host on purpose. Fallback Cloudflare R2 with the `eu` jurisdiction (card requirement only
  community-reported). Hetzner storage has no free tier.
- **Uptime and heartbeat**: Better Stack free (10 monitors, heartbeats, 3-minute checks). UptimeRobot Free has 50
  monitors but no heartbeats; Healthchecks.io (20 heartbeat jobs) is the heartbeat-only alternative.
- **Errors**: Sentry Developer plan, EU region Frankfurt, 5k errors a month, 1 user, 30-day lookback; official
  Angular and NestJS SDKs. GlitchTip (MIT) would need Postgres and 256 to 512 MB on the VPS; not for v1.
- **Analytics**: Cloudflare Web Analytics is free and cookieless by Cloudflare's statement; its default setup excludes
  EU and UK visitors, which must be switched off for this audience.
- **Web hosting**: Cloudflare docs now say "Start new projects with Workers". Workers static assets: static requests
  free and unlimited, 3,000 build minutes, SPA fallback via `not_found_handling`. Pages Free still publishes limits
  (500 builds a month, unlimited previews, Git or Direct Upload chosen once and never swapped). Design moves to
  Workers static assets with Pages as fallback. Apex custom domains need the zone on Cloudflare either way.
- **GitHub**: Actions, public GHCR packages and Dependabot are free for a public repository; GHCR packages start
  private; scheduled workflows auto-disable after 60 idle days.
- **Caddy and VPS**: ports 80/443, an A record and a persistent `/data` volume; Let's Encrypt then ZeroSSL; image
  `caddy:2`. Ubuntu ships unattended-upgrades; Docker-published ports bypass ufw, so only Caddy publishes ports.
- Not verified: R2 card rule officially; Angular prerender plus Cloudflare fallback end to end (phase 0 checks it).
