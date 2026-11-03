# 03: Free transactional email provider for Supabase Auth (facts as of 2026-10-29)

Type: research
Status: resolved
Blocked by: none
Claimed by: research subagent, 2026-10-29

## Question

Which transactional email providers offer, today, a free tier that covers a few hundred auth emails a month
(sign-up confirmation, password reset, magic links) over SMTP as Supabase custom SMTP requires, with EU sending or
data residency, and what do they require (domain verification, card, approval)? What are Supabase's own custom-SMTP
rules and rate limits?

## Answer

Resolved 2026-10-29. Full report with sources: [research/03-transactional-email-provider.md](../research/03-transactional-email-provider.md).

- Supabase custom SMTP: Authentication > Emails > SMTP Settings (host, port, user, password, From). Once enabled,
  Supabase imposes 30 messages an hour, adjustable under Rate Limits. Domain verification is the provider's rule,
  not Supabase's; Supabase recommends a dedicated auth sending domain with SPF, DKIM and DMARC.
- Fit for a few hundred emails a month with 50-a-day bursts, SMTP on the free plan, custom domain, EU: **Mailjet**
  (6,000 a month, 200 a day, EU-only data centres, no card, no approval gate, Starter $9) is the pick, my call.
  SMTP2GO (1,000 a month, 200 a day, Amsterdam, 25 an hour until the domain is verified) and Resend (3,000 a month,
  100 a day, sends from Ireland, account data in the US) are the fallbacks.
- Ruled out: Postmark (100 a month, manual review), Amazon SES (sandbox exit, free plan closes after 6 months),
  Mailtrap (compliance check, no EU), Brevo (deletes free accounts after 4 months unused; figures from search only),
  Mailgun (custom domain on free unresolved between docs and pricing).
- Not verified: Brevo and Scaleway pages blocked the fetch tool; several providers' limits conflict between pages
  (listed in the report).
