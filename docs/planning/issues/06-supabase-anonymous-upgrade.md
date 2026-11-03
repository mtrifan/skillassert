# 06: Does the Supabase anonymous-to-permanent upgrade keep the user id and its rows?

Type: research
Status: resolved
Blocked by: none
Claimed by: research subagent, 2026-10-29

## Question

When an anonymous Supabase user converts to a permanent account (updateUser with email, or linkIdentity with
Google under manual linking), does the auth.users id stay the same so rows keyed by it survive? What happens when
the email or Google identity already belongs to another user? Does the is_anonymous claim flip, and how should RLS
policies treat anonymous users? Sources: Supabase docs and the supabase/auth (GoTrue) source. If the sources are
inconclusive, the fallback is a local spike with the Supabase CLI before the design is approved.

## Answer

Resolved 2026-10-29. Full report with pinned sources: [research/06-supabase-anonymous-upgrade.md](../research/06-supabase-anonymous-upgrade.md).

- The user id is kept on both paths: `updateUser` and `linkIdentity` update the same `auth.users` row (confirmed in
  the Auth server code and, for `updateUser`, an upstream test). Rows keyed by `auth.uid()` survive. The design's
  assumption holds; no `merge_anonymous_user` is needed for the sign-up path.
- Email conversion: the user stays `is_anonymous` until the confirmation link or OTP is used (unless auto-confirm
  is on); the password is stored at the call.
- Google link needs manual linking on; an identity already linked elsewhere comes back as redirect error parameters
  and leaves the anonymous user unchanged. An email already in use returns 422 `email_exists`. No server-side merge
  exists: the returning-learner case needs an app-side merge (round 5 Q43, claim-token design in design.md).
- `is_anonymous` is a boolean claim, role `authenticated`; docs pattern is a restrictive policy on the claim.
- The session survives conversion but the stored JWT keeps the anonymous claim until the next refresh; email
  confirmation and the OAuth callback issue new sessions.
- Defaults: 30 anonymous sign-ins an hour per IP; CAPTCHA strongly recommended. Cleanup: docs say none, the code has
  an off-by-default 30-day cleanup; hosted setting unknown, so the design cleans up itself.
- Open upstream issues to re-test in the Identity phase: auth#1525 (Google link conflict orphans anonymous rows),
  auth#2847 (email upgrade error under default confirm-email config). supabase-js#2512 (PKCE verifier lost on
  refresh) was closed 2026-07-13.
- Nothing was run live; a 10-step local spike recipe is at the end of the report if the Identity phase wants proof.
