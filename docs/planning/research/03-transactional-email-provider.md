# Transactional email providers for skillassert.com (research ticket 03)

- Date researched: 2026-10-29. All lookups were made on that date. URLs marked (search) or "blocked" could not be opened by the fetch tool.
- Markers: "(search)" means the figure came from the search tool's reading of an official page that the fetch tool could not open (HTTP 403 or empty body). "(3rd party)" means a secondary source only.
- Scope: free-tier transactional email behind Supabase Auth custom SMTP (sign-up confirmation, password reset, possibly magic links). Volume: a few hundred emails per month, bursts of about 50 in a day. Developer is in the EU and prefers EU sending and data residency. Budget: free preferred, whole app under EUR 20/month. DNS is on Cloudflare.
- Facts only. No recommendation and no winner.
- Status: complete. Gaps are listed in section 6.

## 2. Summary table

| Provider | Free volume | SMTP on free | Custom domain on free | EU option | Card | First paid step | Source |
|---|---|---|---|---|---|---|---|
| Resend | 3,000/mo, 100/day | Yes | Yes, up to 3; verified domain needed to send beyond the account owner | Sending region Ireland; account data stays in US | Not stated | Pro $20/mo, 50,000 emails | resend.com/pricing |
| Brevo | 300/day | Yes (search) | Domain must be authenticated (search) | EU hosting stated (search) | No (3rd party) | Starter about $9/mo, 5,000 emails (3rd party) | brevo.com/free-smtp-server (search) |
| Postmark | 100/mo, no overage | Listed in features, not tied to Free | Yes; sending beyond own verified domains needs manual approval | No EU servers | No | Basic $15/mo, 10,000 emails | postmarkapp.com/pricing |
| Mailgun | 100/day | Yes | Pricing page says 1; docs FAQ says none; unresolved | EU region selectable; plan availability not stated | Not stated; help center conflicts | Basic from $15/mo, 10,000 emails | mailgun.com/pricing |
| Mailjet | 6,000/mo, 200/day | Yes | Yes; sender or domain validation required | EU-only data centers stated | No at sign-up | Starter $9/mo, 8,000 emails | mailjet.com/pricing |
| SMTP2GO | 1,000/mo, 200/day, 25/hour until domain verified | Yes | Yes; verified sender needed | EU data centre (Amsterdam), EU hostnames | No | Starter $10/mo, 10,000 emails | smtp2go.com/pricing |
| Amazon SES | Pricing page: AWS credits up to $200; FAQ still lists 3,000 message charges/mo for 12 months | Yes | Yes; verified identity; sandbox until production access | Yes, 5 EU SMTP regions | Not stated on pages read | Pay as you go $0.10 per 1,000 emails | aws.amazon.com/ses/pricing |
| Mailtrap | 4,000/mo, 150/day | Yes | Yes, 1 domain, verified before sending | No, US-hosted; EU storage planned for 2026 | No | Basic $15/mo, 10,000 emails | mailtrap.io/pricing |
| MailerSend | 500/mo, 100 API requests/day | Yes | 1 domain | Belgium data center (search) | Yes | Hobby $7/mo, 5,000 emails | mailersend.com/pricing |
| Scaleway TEM | 300 emails free allowance (search) | Yes (3rd party) | Yes (SPF, DKIM) | French provider; region not verified | Not verified | EUR 0.25 per extra 1,000; Scale EUR 80/mo (search) | scaleway.com pricing (blocked) |
| Lettermint | 300/mo | Yes | 1 domain | EU-only stated | No | Starter EUR 10/mo, 10,000 emails | lettermint.co/pricing |
| Postscale | 1,000/mo send and receive | Not mentioned | 1 domain | EU/EEA stated | No | Starter EUR 9/mo, 10,000 emails | postscale.io/pricing |
| Sweego | 100/day | Yes | Unlimited domains | Not stated | Not stated | Individual EUR 8/mo billed annually, 20,000 emails | sweego.io/pricing |
| SendPulse | 12,000/mo, 50/hour | Yes | 2 sender domains | "European data centers" | No | Pay as you go 10,000 emails for $15 | sendpulse.com/prices/smtp |
| Maileroo | 3,000/mo | Yes | 2 domains | Hosted in DE, NL, FI, FR | Not stated | $10/mo, 25,000 emails | maileroo.com/buy-smtp |
| Elastic Email | 100/day | Yes, all plans | Not stated | Not stated | Not stated | Not read | elasticemail.com/smtp |

No recurring free tier, so not in the table: Twilio SendGrid (free trial only), Cloudflare Email Service (arbitrary recipients need Workers Paid), Zoho ZeptoMail (one-time 10,000-email credit). Details are in section 4.

## 3. Supabase custom SMTP facts

Sources (read 2026-10-29): https://supabase.com/docs/guides/auth/auth-smtp, https://supabase.com/docs/guides/auth/rate-limits, https://supabase.com/docs/guides/deployment/going-into-prod (production-checklist wording checked against the docs source file on GitHub).

- **Configuration:** Dashboard, Authentication > Emails > SMTP Settings. Needs SMTP host, port, user, password and a default From address (sender email and sender name). The Management API takes the same fields (`smtp_host`, `smtp_port`, `smtp_user`, `smtp_pass`, `smtp_admin_email`, `smtp_sender_name`). The docs example uses port 587.
- **Built-in mailer:** delivers only to project team addresses (error "Email address not authorized"), 2 emails per hour, best effort with no SLA, described as non-production. The rate-limits page says "2 emails per hour with the built-in email provider".
- **Limit after enabling custom SMTP:** "a low rate-limit of 30 messages per hour is imposed", adjustable under Authentication > Rate Limits. The production checklist words it as "30 new users per hour" and says a major public announcement will likely need more.
- **Limits that still apply:** a new magic link/OTP, sign-up confirmation or password reset for the same user is allowed once per 60 seconds. Sign-ups and sign-ins are limited to 30 requests per 5 minutes per IP (rate-limits page).
- **Sender domain verification:** the Supabase pages set no verification rule of their own. They recommend a custom domain with DKIM, DMARC and SPF set up with the provider, a separate auth sending domain and From address (example `no-reply@auth.example.com`), a second SMTP service on standby, and CAPTCHA. Domain verification is enforced provider-side (see section 4).
- **Link tracking:** with custom SMTP the docs recommend disabling provider link tracking, because it "may overwrite or deform" confirmation links.
- **Plan:** none of the pages read states a plan restriction for custom SMTP.
- **Provider guides listed by Supabase:** Resend, AWS SES, Postmark, Twilio SendGrid, ZeptoMail, Brevo. Any SMTP server works.
- **Alternative to SMTP:** the Send Email Auth Hook (HTTP hook) for custom templating, queuing or several providers.

## 4. Providers

### Resend
- **Free:** 3,000 emails/month, 100/day (UTC day), 3 domains, 30-day logs, SMTP relay yes, first paid is Pro at $20/month for 50,000 emails with no daily cap. https://resend.com/pricing, https://resend.com/docs/knowledge-base/account-quotas-and-limits
- **SMTP:** `smtp.resend.com`, user `resend`, password is the API key. Ports 465/2465 implicit TLS, 25/587/2587 STARTTLS. The Supabase guide uses 465. https://resend.com/docs/send-with-supabase-smtp
- **Domain:** "must add and verify at least one domain to send emails with Resend" (https://resend.com/docs/dashboard/domains/introduction). The resend.dev test sender only delivers to the account owner's address (https://resend.com/docs/knowledge-base/403-error-resend-dev-domain).
- **DNS on Cloudflare:** MX and SPF TXT on the `send` subdomain, DKIM TXT on `resend._domainkey`, kept "DNS Only"; DMARC TXT is optional with `p=none` suggested. https://resend.com/docs/dashboard/domains/cloudflare, https://resend.com/docs/dashboard/domains/dmarc
- **EU:** per-domain sending region Ireland (eu-west-1); account data (metadata, logs, API records) stays in the US. https://www.resend.com/docs/dashboard/domains/regions.md
- **Card, approval, terms:** card not stated and no approval step stated. ToS s.6 lets Resend change or cancel the Free Tier at any time without notice. No inactivity clause. AUP requires opted-in recipients and has no transactional-specific rule. https://resend.com/legal/terms-of-service, https://resend.com/legal/acceptable-use

### Brevo
Brevo's pricing page and help center returned no body or HTTP 403 to the fetch tool, so most figures below are (search) or (3rd party).
- **Free:** 300 emails/day, SMTP relay on every tier, Brevo footer on free emails (3rd party). https://www.brevo.com/free-smtp-server/ (search). First paid: Starter from about $9/month for 5,000 emails (3rd party 2026 roundups). Card not required (3rd party).
- **SMTP:** `smtp-relay.brevo.com:587` (3rd party). Supabase links a Brevo guide: https://help.brevo.com/hc/en-us/articles/7924908994450
- **Domain and DNS:** authentication is mandatory, citing the Gmail and Yahoo rules of 2024-02-01. An unauthenticated domain is rewritten to `@brevosend.com`, and Gmail-type sender addresses cannot be authenticated. Records: "Brevo code" TXT, DKIM (TXT or two CNAMEs), DMARC TXT; automatic setup through a DNS-provider login is offered. https://help.brevo.com/hc/en-us/articles/12163873383186 (search)
- **EU:** help center says all servers are in the EU (OVH in France and Germany, Google Cloud in Belgium). https://help.brevo.com/hc/en-us/articles/360001005510 (search)
- **Approval:** 3rd-party guides say a new account needs manual activation for transactional SMTP after domain verification. No official page read.
- **Inactivity:** Free accounts are deleted after 4 months without use, with reminders at 30, 15, 7 and 1 days; transactional emails count as activity. https://help.brevo.com/hc/en-us/articles/4410311028626 (search). Terms of Use dated 2025-10-01, s.3.4, give 6 months. https://www.brevo.com/termsofuse/ (search)

### Postmark
- **Free:** Developer plan, 100 emails/month, "No overages allowed", "No Credit Card Required". SMTP is in the feature list but not explicitly tied to Free. First paid: Basic $15/month from 10,000 emails, $1.80 per extra 1,000. https://postmarkapp.com/pricing
- **Approval:** until a manual review passes (under 24 hours on weekdays) you can send only to domains you added and verified; link tracking is off until approval. https://postmarkapp.com/support/article/1084-how-does-the-account-approval-process-work
- **DNS on Cloudflare:** DKIM TXT (host and value from DNS Settings) and a Return-Path CNAME to `pm.mtasv.net` with the proxy off. The page does not say which record is mandatory and does not mention DMARC. https://postmarkapp.com/support/article/adding-dkim-and-return-path-records-to-cloudflare
- **EU:** none. Data sits in a Chicago-area data center and AWS, and "We currently don't have plans to add servers in the EU". DPA with Standard Contractual Clauses. https://postmarkapp.com/eu-privacy
- **Terms:** no inactivity clause ("Inactivity does not constitute automatic cancellation", s.10(a)). Postmark may terminate an account at any time (s.15). No transactional-only restriction; lists must be permission-based (s.5(c)). https://postmarkapp.com/terms-of-service

### Mailgun
- **Free:** 100 emails/day, 1 custom domain, 1-day logs, API and SMTP relay, 1 user. Basic from $15/month for 10,000 emails; Foundation $35/month for 50,000. https://www.mailgun.com/pricing/
- **Custom domain on Free, unresolved:** the pricing page lists 1 custom domain and never mentions a card. The docs FAQ says "Free accounts do not include the ability to create a custom domain" (https://documentation.mailgun.com/docs/mailgun/faq/faqs). The sandbox domain sends only to authorized recipients, lists up to 5 (https://documentation.mailgun.com/docs/mailgun/user-manual/domains/domains-sandbox). Help-center articles (search) say Free needs no card and allows 1 custom domain (https://help.mailgun.com/hc/en-us/articles/203068914-What-does-the-Free-plan-offer, https://help.mailgun.com/hc/en-us/articles/203637190-How-can-I-add-or-delete-a-domain), yet that Free accounts without card details are limited to authorized recipients (https://help.mailgun.com/hc/en-us/articles/217531258-Authorized-Recipients).
- **EU:** "you choose which region your data is processed in" and message data "never leaves the region" (https://www.mailgun.com/about/regions/). SMTP hosts are `smtp.mailgun.org` and `smtp.eu.mailgun.org` (search of Mailgun docs). Plan availability of the EU region is not stated.
- **DNS:** SPF TXT `v=spf1 include:mailgun.org ~all`, DKIM TXT, tracking CNAME to `mailgun.org`, MX 10 `mxa.mailgun.org` and `mxb.mailgun.org`; the page marks all four required. EU values, DMARC and Cloudflare are not covered. https://documentation.mailgun.com/docs/mailgun/user-manual/domains/domains-verify
- **Terms:** use limited to "commercial and professional purposes only" (s.3.1). Neither Terms nor AUP has an inactivity clause. AUP allows one free account per platform (s.4) and exempts transactional mail from the permission rule (s.1c). https://www.mailgun.com/legal/terms/, https://www.mailgun.com/legal/aup/

### Mailjet
- **Free:** 6,000 emails/month, 200/day, SMTP relay, 1 subaccount, "no credit card required for the free plan at sign up". "No Mailjet Logo" is listed from Essential up, which implies a logo on Free. First paid: Starter $9/month for 8,000 emails, no daily cap. https://www.mailjet.com/pricing/
- **SMTP:** `in-v3.mailjet.com`, ports 587/588/25/80/2525 (TLS optional) and 465 (SSL); user is the API key and password the secret key. https://dev.mailjet.com/smtp-relay/configuration/
- **Domain:** a sender address or domain must be added and validated before sending. SPF and DKIM "will need" setup; DMARC is not mentioned; record values are not on that page. https://dev.mailjet.com/email/guides/senders-and-domains/
- **EU:** "secure data centers located exclusively within the European Union", country not named there. https://www.mailjet.com/products/data-security-and-privacy/
- **Inactivity:** help-center policy (search): reminders at 2 and 6 months, re-engagement at 9 months, 30-day deletion notice at 12 months, final notice at 400 days. Applicability to Free accounts is not confirmed. https://documentation.mailjet.com/hc/en-us/articles/36935583263131. The terms page returned 404.

### SMTP2GO
- **Free:** 1,000 emails/month, 200/day, and 25/hour until the sender domain is verified. After the monthly cap, mail is rejected, not queued. The plan does not expire. No card. First paid: Starter $10/month (or $100/year), 10,000 emails. https://www.smtp2go.com/pricing/, https://www.smtp2go.com/faq, https://www.smtp2go.com/blog/know-your-sending-limits/
- **SMTP:** `mail.smtp2go.com`, ports 25/80/587/2525/8025 (TLS) and 465/8465/443 (SSL). Needs an SMTP user and at least one verified sender. https://developers.smtp2go.com/docs/smtp-relay
- **DNS:** three CNAMEs shown in the dashboard (return-path, link tracking, DKIM), no SPF change, up to 48 hours to verify (search; the support site returned 403).
- **EU:** EU sign-ups are handled by the Amsterdam data centre. `mail-eu.smtp2go.com` (EU and UK) or `mail-eu2.smtp2go.com` (EU only) keep traffic in the EU. Free-plan access to this is not stated. https://www.smtp2go.com/blog/sending-emails-eu-youll-want-read
- **Terms:** free plans are "primarily for trial purposes and for sending low amounts of email"; at most 10 free accounts per business; no inactivity clause. https://www.smtp2go.com/terms/

### Amazon SES
- **Free, conflicting:** the pricing page has no SES allowance and cites only AWS Free Tier credits for new customers, up to $200 ($100 at sign-up plus up to $100 earned). The SES FAQ still says "Receive up to 3,000 message charges for free each month for the first 12 months", in the same section that says credits apply to new customers from 2025-07-15, and does not say which applies. Pay as you go is $0.10 per 1,000 emails plus $0.12/GB of attachments; bundles start at $0.16 per 1,000. https://aws.amazon.com/ses/pricing/, https://aws.amazon.com/ses/faqs/
- **AWS Free plan:** ends after 6 months or when credits run out. The account then closes, content is kept 90 days, and an upgrade to a Paid plan is needed. Credits expire after 12 months. The pages read state no card requirement and do not say whether SES is excluded. https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html, https://aws.amazon.com/free/terms
- **Sandbox:** each region starts in a sandbox: only verified addresses or domains, 200 messages per 24 hours, 1 per second. Production access is a console request (mail type, website URL, acknowledgement) with a first response within 96 hours. The FAQ still describes a Support Center sending-limit-increase request instead. https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html
- **SMTP:** credentials are separate from IAM keys and per region; TLS only; 25/587/2587 STARTTLS, 465/2465 TLS wrapper. EU SMTP endpoints exist in Frankfurt, Ireland, London, Paris and Stockholm, not Milan or Zurich. https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html, https://docs.aws.amazon.com/general/latest/gr/ses.html
- **DNS:** Easy DKIM is three CNAME records per region, up to 72 hours to verify; a custom MAIL FROM (MX plus SPF TXT) is optional; no DMARC statement on the page. https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html

### Mailtrap (Email Sending)
- **Free:** 4,000 emails/month, 150/day, 1 domain, 3-day logs, "API and SMTP relay", no card. First paid: Basic $15/month for 10,000 emails and 5 domains. https://mailtrap.io/pricing/
- **Domain:** only Verified domains can send; the demo domain sends only to the registered address. Records: four CNAMEs (verification, two DKIM, custom tracking) plus a DMARC TXT; no SPF record is listed. An automatic compliance check follows, and some domains must submit a compliance form or pass manual review. https://docs.mailtrap.io/email-api-smtp/setup/sending-domain
- **EU:** data is hosted in the US (AWS us-east-1 and Google US). "We plan to offer EU-based data storage in 2026." https://docs.mailtrap.io/account-and-organization/privacy-and-security/gdpr-compliance
- **Terms:** "MAILTRAP IS OFFERED ONLY FOR BUSINESS OR PROFESSIONAL USAGE"; may log off users inactive "for an extended period"; may suspend at any time; the free plan is for one user. https://mailtrap.io/terms/ The commercial-use notice at https://mailtrap.io/mailtrap-use/ concerns the Email Testing free plan, not Sending.

### Other providers checked
- **MailerSend:** Free 500 emails/month, 100 API requests/day, 1 domain, 1-day activity data, API and SMTP relay. A card is required to subscribe "to prevent abuse of the platform", not charged. Hobby $7/month for 5,000. Data center in Belgium on Google Cloud (search). https://www.mailersend.com/pricing, https://www.mailersend.com/legal/how-mailersend-stays-gdpr-compliant
- **Scaleway TEM (FR):** official pages returned 403. Search reading of its pricing: Essential includes 300 emails per organization free, then EUR 0.25 per extra 1,000; Scale is EUR 80/month for 100,000. Docs list SPF include `_spf.tem.scaleway.com`, DKIM, MX recommended and DMARC, and disagree on whether MX and DMARC are mandatory. https://www.scaleway.com/en/pricing/managed-services/, https://www.scaleway.com/en/docs/transactional-email/how-to/authenticate-domain/
- **Lettermint (NL):** Free 300 emails/month, SMTP relay, 1 domain, no card, EU-only data. Starter EUR 10/month for 10,000. https://lettermint.co/pricing
- **Postscale (EE):** Free 1,000 emails/month send plus receive, 1 domain, no card, production data in the EU/EEA, SMTP relay not mentioned. Starter EUR 9/month for 10,000. https://postscale.io/pricing
- **Sweego (FR):** Free 100 emails/day by API and SMTP, unlimited domains; card and data location not stated. Individual EUR 8/month billed annually (EUR 10 monthly) for 20,000. https://www.sweego.io/pricing
- **SendPulse SMTP:** Free 12,000 emails/month, 50/hour, 2 sender domains, no card, "European data centers". Pay as you go 10,000 emails for $15, credits valid 12 months. https://sendpulse.com/features/smtp, https://sendpulse.com/prices/smtp
- **Maileroo:** Free 3,000 outbound plus 1,000 inbound emails/month, 2 domains, SMTP relay, hosted in Germany, Netherlands, Finland and France; card not stated; $10/month for 25,000. https://maileroo.com/, https://maileroo.com/buy-smtp
- **Elastic Email:** "100 emails/day free", SMTP relay on all plans. Domain rules, card, EU and paid prices not read. https://elasticemail.com/smtp
- **Zoho ZeptoMail (one-time credit):** first credit is free, 10,000 emails. The pricing page says credits last 6 months; Zoho FAQ pages (search) say the sign-up credit lasts 1 month. Cap is 100/day until the account passes review (validation form, about 2 business days). SMTP `smtp.zeptomail.com`, user `emailapikey`; domain verified by DKIM plus a return-path CNAME; data centers US, EU and IN. https://www.zoho.com/zeptomail/pricing.html, https://www.zoho.com/zeptomail/help/getting-started.html, https://www.zoho.com/zeptomail/help/smtp-home.html
- **Cloudflare Email Service:** sending is beta with SMTP supported. Arbitrary recipients need Workers Paid (3,000 emails/month included, then $0.35 per 1,000); the Free plan sends only to verified destination addresses. https://developers.cloudflare.com/email-service/, https://developers.cloudflare.com/email-service/platform/pricing/
- **Twilio SendGrid:** page offers a free trial only ("No credit card required") and no free plan. https://www.twilio.com/en-us/sendgrid

## 5. Implications for skillassert

Facts and constraints only. No ranking.
- **Supabase side:** the custom SMTP default is 30 messages/hour, adjustable. A 50-email burst inside one hour exceeds it until the limit is raised. A second magic link, reset or confirmation for the same user is refused for 60 seconds.
- **Daily caps against a 50-email burst:** Resend 100, Mailgun 100, Sweego 100, Elastic Email 100, Mailtrap 150, Mailjet 200, SMTP2GO 200, Brevo 300. Hourly caps: SMTP2GO 25 until the domain is verified, SendPulse 50.
- **Monthly allowances against "a few hundred":** Postmark 100 (below), Lettermint 300, Scaleway 300, MailerSend 500, SMTP2GO 1,000, Postscale 1,000, Resend 3,000, Maileroo 3,000, Mailtrap 4,000, Mailjet 6,000, SendPulse 12,000. Mailgun's free cap is daily only. SES has no allowance on its pricing page.
- **Sender domain:** every provider that sends to arbitrary users needs skillassert.com or a subdomain verified in DNS. Record counts: Postmark 2, Resend 3, SMTP2GO 3, SES 3 (plus 2 optional), Brevo 3 to 4, Mailgun 4 types, Mailtrap 5. Mailjet's values were not on the page read.
- **Cloudflare:** Resend and Postmark say to keep their DKIM or Return-Path records unproxied (DNS only). Resend notes Cloudflare error 1004 on proxied CNAMEs.
- **DMARC stated by providers:** Brevo mandatory, Mailtrap lists a DMARC TXT among five records, Resend optional, Scaleway docs conflict. The Postmark, Mailgun, Mailjet, SES and SMTP2GO pages read state no DMARC rule.
- **Approval lead time:** Postmark manual review under 24 hours on weekdays; SES production access with a first reply within 96 hours; Mailtrap automatic compliance check, sometimes manual; ZeptoMail review about 2 business days; Brevo manual activation (3rd party). None stated for Resend, Mailjet, SMTP2GO; Mailgun unresolved.
- **Recipient limits before verification or approval:** Resend (resend.dev only to the owner), SES sandbox (verified recipients only), Postmark (own verified domains only), Mailgun sandbox (authorized recipients), Mailtrap demo domain (registered address), Cloudflare Free (verified destinations).
- **EU residency:** EU-only data stated by Mailjet, Lettermint, Postscale, Maileroo, SendPulse, plus Brevo and MailerSend (search). Region choice: Mailgun, SMTP2GO, SES. Split: Resend (Ireland sending, US account data). None: Postmark, Mailtrap (EU storage planned for 2026).
- **Inactivity and expiry:** Brevo deletes Free accounts after 4 months without use (terms say 6; transactional email counts as activity). Mailjet's policy ends in deletion after 12 months or more. The AWS Free plan closes after 6 months unless upgraded. SMTP2GO says its free plan does not expire. No inactivity clause was found in the Resend, Postmark, Mailgun, SMTP2GO or Mailtrap terms read.
- **Terms on use:** Mailgun and Mailtrap limit use to business or professional purposes. SMTP2GO calls free plans "primarily for trial purposes". Resend may change or cancel its Free Tier without notice. No terms read forbid transactional or auth email on a free plan.
- **Cards:** required by MailerSend. Stated as not required by Postmark, Mailjet, SMTP2GO, Mailtrap, Lettermint, Postscale and SendPulse. Not stated or unresolved: Resend, Mailgun, SES, Sweego, Maileroo, Elastic Email.
- **Link tracking:** Supabase advises turning provider link tracking off. Postmark enables tracking only after approval. Mailgun, SMTP2GO and Mailtrap ask for a tracking CNAME.
- **Budget:** first paid steps are in USD for Resend ($20), Postmark ($15), Mailgun ($15), Mailtrap ($15), SMTP2GO ($10), Mailjet ($9), MailerSend ($7) and in EUR for Lettermint (10), Postscale (9), Sweego (8). SES is per use. The budget is EUR 20/month and no currency conversion was applied.

## 6. Could not verify

- **Brevo:** pricing page, help center and terms were unreadable to the fetch tool. The 300/day limit, EU hosting, mandatory domain authentication and the 4-month and 6-month inactivity rules are search readings. The Starter price, the footer on free emails, the SMTP host and port, and manual SMTP activation come from 3rd-party pages only.
- **Mailgun:** whether a Free account can add a custom domain and send to arbitrary recipients without a card. Pricing page, docs FAQ and help center disagree. EU DNS values and EU availability on Free were not found.
- **Amazon SES:** whether a new account gets the 3,000-message free allowance (FAQ yes, pricing page no), whether a card is needed for the Free plan, and whether SES works on a Free account plan.
- **Mailjet:** SPF and DKIM record values, the terms of use (404), whether the inactivity policy covers Free accounts, and the logo on Free.
- **SMTP2GO:** "up to 5 verified senders" on Free appears only in a search reading. EU data centre access on Free and the exact CNAME targets were not confirmed.
- **Resend and Postmark:** Resend card requirement is not stated. Postmark SMTP on the Free plan is implied by the feature list, not stated.
- **Scaleway TEM, Elastic Email, Sweego, Maileroo:** official pages were blocked or silent on card, EU region or paid prices. Scaleway terms were not read.
- **ZeptoMail:** free credit validity is 6 months on the pricing page and 1 month in Zoho FAQ pages (search).
- **Cloudflare Email Service:** the Workers Paid base price was not on the pages read.
- **Seen only in competitor blogs, not read officially:** Mailofly, Sequenzy, Plunk, Nitrosend. AhaSend, Emailit and MailPace returned nothing.
- **Method caveat:** the fetch tool summarizes pages with a small model, so figures may carry transcription errors. The Supabase 30-per-hour wording was checked against the raw docs source on GitHub.
