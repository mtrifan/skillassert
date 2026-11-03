# 06 - Supabase anonymous-user upgrade semantics

- Date: 2026-10-29. Scope: facts with citations, no recommendation. Primary sources only (Supabase docs source, supabase/auth, supabase-js/auth-js, official GitHub issues and discussions).
- Confidence words: confirmed in code, stated in docs, reported in issue, not found. Nothing was run against a live Supabase stack; "confirmed in code" means read from source, plus an upstream unit test where noted.
- Citation key. Every URL is commit-pinned; a cite like `auth:internal/api/user.go#L112-L118` means key URL plus that path.
  - `auth:` https://github.com/supabase/auth/blob/fcc491d6d7665718b62f50598d723fde1e9b6c4a/ (HEAD on 2026-10-29). In the files cited here, the latest release v2.197.0 (2026-09-09) differs from HEAD only in unrelated edits to external.go, mail.go and verify.go.
  - `js:` https://github.com/supabase/supabase-js/blob/ecab7fb696e331b460205f233c76ef94a2e36c85/packages/core/auth-js/src/GoTrueClient.ts (auth-js now lives in supabase-js; supabase/auth-js is archived).
  - `docs:` https://github.com/supabase/supabase/blob/7d6671e6ea1fddd48a5add65746d4981e400d3b4/apps/docs/content/guides/auth/ (rendered at https://supabase.com/docs/guides/auth/<page>).

## 1. Verdict as facts

- Q1 updateUser({email, password}): same `auth.users.id`, row updated in place. `is_anonymous` becomes false only when the email change is confirmed, or inside the same request if the project auto-confirms email. The password hash is stored at the updateUser call, before confirmation. Confirmed in code and by upstream test. Docs show a different two-call sequence.
- Q2 linkIdentity google, manual linking on: same id. The identity is attached to the user id saved in the OAuth flow state, and `is_anonymous` becomes false in the callback if the provider email is verified. Identity already linked to another user: redirect to the app with `error_code=identity_already_exists`, "Identity is already linked to another user"; transaction rolled back, anonymous user unchanged. Confirmed in code; no upstream test covers the anonymous variant.
- Q3 email already used: updateUser returns 422 `email_exists`. A Google link whose email belongs to another user returns 400 `email_exists` via the redirect. No server-side merge exists; the app moves rows itself. Confirmed in code; docs say the app resolves conflicts.
- Q4 claim: `is_anonymous` is a JSON boolean, always present. Anonymous users have role `authenticated`. Docs policy: `(select (auth.jwt()->>'is_anonymous')::boolean) is false`. Confirmed in code and stated in docs.
- Q5 session: the same session and refresh token stay valid through updateUser. The stored access token is not reissued, so its claim stays true until the next refresh, which builds claims from the current DB row. Email confirmation and the OAuth link callback each create a new session for the same user id. Confirmed in code.
- Q6 abuse: default 30 anonymous sign-ins per hour per IP, adjustable. CAPTCHA strongly recommended. Docs say there is no automatic cleanup and give a 30-day SQL; code contains an off-by-default 30-day cleanup. Stated in docs and confirmed in code. Hosted cleanup setting: not found.
- Q7 issues: no report of an id change or of rows deleted by the conversion itself. Open and related: auth#1525 (Google link fails when the identity exists elsewhere, anonymous rows orphaned) and auth#2847 (email upgrade error under default confirm-email config, unverified). Reported in issues.

## 2. Details

### Q1 updateUser on an anonymous user
- Handler `auth:internal/api/user.go#L82-L305` modifies the authenticated user's row and inserts no user. Upstream test `auth:internal/api/anonymous_test.go#L87-L216` asserts: still anonymous after PUT /user (L165), same id after verify (L197), `is_anonymous` false (L199), one identity, app_metadata providers `["email"]`.
- Password alone on an anonymous user is rejected: 422 `validation_failed`, "Updating password of an anonymous user without an email or phone is not allowed" (`user.go#L112-L118`). Email plus password in one call is allowed since PR #1739 (merged 2024-08-26; its text: "An anonymous user remains anonymous until they have confirmed the email / phone added").
- The password hash is written in the same request (`user.go#L206-L237`). `users.email` stays empty and the address waits in `email_change`, so password sign-in by that email cannot work before confirmation (inferred from `auth:internal/models/user.go#L564-L592`, which sets email only on confirm).
- Confirm-email ON (hosted default per SDK docs, `js#L871`): the mail goes to the new address only (`user.go#L263-L275`, `auth:internal/mailer/templatemailer/templatemailer.go#L257-L279`). The two-address secure change needs an existing email (`auth:internal/api/verify.go#L548-L551`). On confirm, `verify.go#L546-L639` creates the `email` identity (L594-L605), sets `is_anonymous=false` (L614-L619) and sets email (L623) in one transaction, then issues a new session (`verify.go#L185`, `#L285`).
- Confirm-email OFF (autoconfirm): `user.go#L251-L262` runs the same confirm routine inside the PUT, so the user is permanent when the call returns (PR #1646, merged 2024-07-04).
- Docs (`docs:auth-anonymous.mdx#L125-L159`) show updateUser({email}), verify, then updateUser({password}), and say the email must be verified before a password is added. The combined call is not mentioned there.
- Docs say conversion "requires you to enable manual linking" (`auth-anonymous.mdx#L127`). In server code that flag guards only `/user/identities/*` (`auth:internal/api/api.go#L285-L289`, `auth:internal/api/middleware.go#L396-L402`); `PUT /user` does not check it. Not exercised on hosted.
- Column: `is_anonymous boolean not null default false` (`auth:migrations/20240214120130_add_is_anonymous_column.up.sql`). Anonymous sign-in creates no identity row and sets no provider in app_metadata (`auth:internal/api/anonymous.go#L12-L62`, `auth:internal/api/signup.go#L97-L104`).

### Q2 linkIdentity google with manual linking
- Client calls GET `/user/identities/authorize` with the access token and `skip_http_redirect=true`, then navigates the browser to the returned URL (`js#L4681-L4720`). Manual linking is off by default (`auth:internal/conf/configuration.go#L934`); when off the route returns 404 `manual_linking_disabled`.
- The server stores the current user id as the linking target in the flow state (`auth:internal/api/external.go#L118-L121`). The callback reloads that user (`external.go#L587-L596`) and `linkIdentityToUser` (`auth:internal/api/identity.go#L173-L237`) inserts the identity for that user. No new user row. A new session for the same id is then issued: in the callback for the implicit flow (`external.go#L251`), at the code exchange for PKCE (`auth:internal/api/token.go#L245-L289`).
- In linkIdentity, `is_anonymous` flips inside the callback transaction, and only if the user had no email and the provider email is verified (`identity.go#L199-L222`). If the provider email is unverified the identity stays linked, a confirmation mail is sent, the call ends with `email_not_confirmed`, and the user stays anonymous (`identity.go#L206-L211`).
- Identity already linked to another user: `identity.go#L181-L186` returns 422 `identity_already_exists`, "Identity is already linked to another user" (same user: "Identity is already linked", unit test `auth:internal/api/identity_test.go#L81-L125`). A plain error rolls the transaction back, so the anonymous user keeps its id, flag and rows, gains no identity, and keeps its session. The error reaches the app as a redirect with `error`, `error_code`, `error_description` in query and fragment (`external.go#L822-L893`; 422 maps to `error=server_error`). `linkIdentity()` itself returns no error here (discussion https://github.com/orgs/supabase/discussions/27061).
- After the redirect, current auth-js returns the error and does not clear the stored session (`js#L691-L711`). Older clients cleared it (https://github.com/supabase/auth/issues/2024, closed; comments in https://github.com/supabase/auth/issues/1525 from 2024-08 and 2024-11). The release that changed it was not determined.
- Docs show only the call (`docs:auth-anonymous.mdx#L239-L259`) and do not name the error code. SDK remark: "If the candidate identity is already linked to the existing user or another user, `linkIdentity()` will fail" (`js#L4651`).
- Maintainer position (auth#1525, 2024-04-14): two auth users cannot share one Google identity; do a normal Google sign-in instead; the anonymous account's activity is lost unless the app moves it.

### Q3 Email already belongs to another user
- updateUser: `user.go#L134-L140` using `IsDuplicatedEmail` (`auth:internal/models/user.go#L927-L972`) returns 422 `email_exists`, "A user with this email address has already been registered" (`auth:internal/api/errors.go#L20`). Not obfuscated and independent of confirm settings.
- linkIdentity where Google's email is another user's email: `models/user.go#L279-L355` returns a unique-conflict, mapped to 400 `email_exists` (`identity.go#L199-L205`), rolled back, delivered via redirect.
- signInWithOAuth (not linking) with a verified email matching an existing user attaches the Google identity to that existing user (`external.go#L302-L341`). The client gets the existing user's session; the anonymous user and its rows stay separate.
- No server-side merge: non-test Go code has no merge or transfer function, and admin routes are only audit log, users CRUD, factors, passkeys, generate_link, sso, oauth (`api.go#L360-L420`).
- Docs "Resolving identity conflicts" (`auth-anonymous.mdx#L343-L401`): the app chooses overwrite, keep or merge. The sample catches the updateUser error, calls signInWithPassword, then runs a client-side `.update({user_id: existingUser.id}).eq('user_id', anonId)`.

### Q4 is_anonymous claim, RLS, role
- Claim built at `auth:internal/tokens/service.go#L713-L732`; field `IsAnonymous bool json:"is_anonymous"` has no omitempty (`#L89`). Docs list it as `boolean` among claims "always present" (`docs:jwt-fields.mdx#L20-L35`).
- Role comes from the JWT default group name in `signupNewUser` (`signup.go#L380-L392`, called at `auth:internal/api/anonymous.go#L43`); `GOTRUE_JWT_DEFAULT_GROUP_NAME="authenticated"` (`auth:example.env#L7`). Docs: "An anonymous user assumes the `authenticated` role like a permanent user" (`auth-anonymous.mdx#L13`, `#L322`).
- Docs policies (`#L324-L333`): restrictive insert policy `to authenticated with check ((select (auth.jwt()->>'is_anonymous')::boolean) is false)` plus a permissive select `using (true)`. Docs warn that permissive policies combine with OR and that a lone restrictive policy fails (`#L336-L339`), and ask readers to review existing policies before enabling anonymous sign-ins (`#L25-L31`).
- Docs describe the anon API key as the "anonymous" Postgres role while JWT docs name it `anon` (`jwt-fields.mdx#L68`). The anon key creates no user.
- Anonymous tokens are refused on MFA enrollment and passkey registration (`api.go#L299-L300`, `#L334`).

### Q5 Session continuity
- Client updateUser PUTs `/user` with the current access token, then saves the same session with `session.user` replaced and emits USER_UPDATED; tokens are not refreshed (`js#L3445-L3462`). The stored JWT still says `is_anonymous: true`; the returned user object shows the new flag.
- Refresh (`auth:internal/tokens/service.go#L194-L283`, `#L598-L603`) loads the user via the refresh token and builds claims from the current row (`#L729`). Conversion does not modify the current session, so the same refresh token keeps working and the next refresh returns false. `refreshSession()` forces it (`js#L3797`). Default JWT lifetime 3600 s, max 604800 s (https://github.com/supabase/cli/blob/bcda9df33f6e9235d68d6ce4178b02902403f1a5/apps/cli/src/shared/init/project-init.templates.ts#L168-L169). The stale-claim window is therefore up to one JWT lifetime (derived).
- Setting a password logs out all other sessions of that user but keeps the current one (`auth:internal/models/user.go#L434-L464`, called at `auth:internal/api/user.go#L214`).
- Email confirmation and the OAuth link each create a new session (`tokens/service.go#L891`; `verify.go#L185`, `#L285`; `external.go#L251`; `token.go#L265`). If the mail link opens in another browser, that browser receives the session under the implicit flow; under PKCE the exchange needs the verifier stored in the originating browser (`js#L2055`, `#L4007-L4026`). Conversion does not revoke the original session, except when "single session per user" is on, where refreshing the older session fails with `session_expired` (`tokens/service.go#L316-L365`).
- Client caveat: https://github.com/supabase/supabase-js/issues/2512 (closed 2026-07-13) reports `_saveSession` deleting the PKCE verifier on every token refresh (2.86.1 to 2.110.2), breaking the PKCE email-link exchange for anonymous-to-email conversion while the server still converted the user at /verify. At HEAD `_saveSession` no longer starts with that deletion (`js#L5288-L5318`).

### Q6 Rate limits, CAPTCHA, cleanup
- Anonymous sign-in is `POST /auth/v1/signup` with no email or phone. Disabled by default (`configuration.go#L90`); disabled returns 422 `anonymous_provider_disabled` (`api.go#L229-L246`). The route passes the CAPTCHA middleware when CAPTCHA is on (`api.go#L229`); the SDK sends `gotrue_meta_security.captcha_token` (`js#L825-L834`).
- Limit key is client IP, or `Sb-Forwarded-For` when enabled. Default 30 per hour, bucket capacity equal to the limit, 429 `over_request_rate_limit` (`middleware.go#L123-L133`, `auth:internal/api/apilimiter/apilimiter.go#L192-L195`, `configuration.go#L490`).
- Docs: "An IP-based rate limit is enforced at 30 requests per hour which can be modified in your dashboard" and "strongly recommended to enable invisible CAPTCHA or Cloudflare Turnstile" (`auth-anonymous.mdx#L405`). Value 30: https://github.com/supabase/supabase/blob/7d6671e6ea1fddd48a5add65746d4981e400d3b4/packages/shared-data/config.ts#L116-L122. Server-side callers need a secret key and a project setting to forward the end-user IP (`docs:rate-limits.mdx#L49-L53`).
- Cleanup, docs: "Automatic cleanup of anonymous users is currently not available", with SQL deleting `is_anonymous is true and created_at < now() - interval '30 days'` (`auth-anonymous.mdx#L407-L415`).
- Cleanup, code: PR #1497 (2024-03-27) added the same rule (`auth:internal/models/cleanup.go#L67-L72`): up to 100 rows per run, one round-robin statement per successful POST/PUT/PATCH/DELETE (`middleware.go#L436-L461`). It runs only if `GOTRUE_DB_CLEANUP_ENABLED` is true, default false (`configuration.go#L141`, `api.go#L193-L196`). The clock is `created_at`, not last activity. Whether hosted projects enable it: not found.

### Q7 Issues and discussions (supabase/auth, supabase/supabase, supabase/supabase-js, archived auth-js; searched 2026-10-29)
- https://github.com/supabase/auth/issues/1525 open, 22 comments, last 2025-09: anonymous to Google link fails with "Identity is already linked to another user"; maintainers call it expected; anonymous activity is not merged.
- https://github.com/supabase/auth/issues/1762 open: same scenario filed as a bug; a commenter calls it a feature request.
- https://github.com/supabase/auth/issues/2847 open, 2026-10-01, no replies: updateUser({email}) on an anonymous user with autoconfirm off fails with `email_address_invalid` and `Email address "" is invalid`. Code shows the message prints the user's current, empty email (`auth:internal/api/mail.go#L768-L771`, `#L925-L928`). Whether valid addresses are really rejected is unconfirmed. Mailer email validation (format, DNS) is off by default in server code (`auth:internal/conf/configuration.go#L690-L693`, `auth:internal/mailer/validateclient/validateclient.go#L177-L180`); the hosted setting was not found.
- https://github.com/supabase/auth/issues/2024 closed 2025-05-14: client signed the user out after `identity_already_exists`; reporter says it was fixed. https://github.com/supabase/supabase-js/issues/2512 closed 2026-07-13: see Q5.
- https://github.com/supabase/auth/issues/1619 and https://github.com/supabase/auth/issues/1671 closed: autoconfirm ignored for anonymous email, and no password for anonymous users; fixed by PRs #1646 and #1739.
- https://github.com/supabase/auth/issues/1795 open: new anonymous user on every app start when the session is not persisted (Flutter). Discussion https://github.com/orgs/supabase/discussions/37797: after Google linking, `user_metadata` may hold only `email_verified` until next sign-in (reported, not verified).
- Not found: any issue reporting that `auth.users.id` changes, or that the conversion itself deletes rows, for updateUser or linkIdentity.

## 3. Implications for skillassert (facts and constraints, no recommendation)

- Rows keyed by the auth user id need no migration when the same user converts by either path (Q1, Q2). The id is the one `signInAnonymously` returned.
- With confirm-email on, the user is `is_anonymous=true` with no `users.email` until the link or OTP is used. Any policy or UI gate on `is_anonymous` treats them as anonymous for that whole period.
- A freshly converted session carries an access token that still says anonymous until refresh, so a permanent-only RLS policy can deny it for up to one JWT lifetime unless the client refreshes (derived from Q5).
- Google path: conversion completes in the OAuth callback; failures arrive only as redirect parameters, so the page that receives the redirect must read them.
- `signInWithOAuth` on an anonymous session does not convert, because the plain authorize route passes no linking target (`auth:internal/api/external.go#L27-L34`). It signs in as a different user (new, or the existing one with the same verified email) and the anonymous user row stays separate.
- In both conflict cases the anonymous id and its rows survive but become orphaned once the visitor signs in as the existing user, because `auth.uid()` is then the other id (derived). The docs sample reassigns rows from the client after sign-in; under owner-only RLS that update would match no rows (derived, untested). The anonymous id must be captured before the sign-in.
- Stale-user removal is the docs SQL or the off-by-default server cleanup, both 30 days from `created_at`. If app tables reference `auth.users(id)` with `on delete cascade`, a cleanup run would delete attempts, bookmarks and flags of anonymous users older than 30 days. The FK action is the app's choice.
- The anonymous identity lives in browser storage. Clearing storage or switching device yields a new anonymous user and orphaned rows (`auth-anonymous.mdx#L11`). Calling `signInAnonymously()` on each load without reusing a session creates a user per call and spends the per-IP quota (derived from `anonymous.go#L12-L62`).
- The limit is per IP. Calls made from a server or edge function share that server's IP unless IP forwarding is enabled with a secret key; offices and NAT share a bucket.
- Settings off by default in server code and CLI: anonymous sign-ins, manual linking. Google needs provider credentials. Hosted "Confirm email" is on by default per SDK docs.
- Anonymous users hold the same `authenticated` role, so every table policy that should exclude them needs an explicit `is_anonymous` check, using restrictive policies per the docs.

## 4. Could not verify

- Hosted-platform specifics: whether cleanup is enabled, hosted defaults for manual linking, and the exact auth server version in the hosted fleet. Documented behavior matches the latest release tag v2.197.0.
- No live run. The Supabase CLI is not installed here. No upstream automated test covers anonymous plus linkIdentity.
- auth#2847: whether valid real addresses fail under default confirm-email.
- Which auth-js release stopped clearing the session on `identity_already_exists`.
- Whether the docs claim that email conversion requires manual linking holds on hosted, since server code does not check it.
- Send Email hook payload for anonymous conversion: auth#2042 was closed as fixed by PR #2044 per a maintainer comment, not checked in code.
- Id retention is confirmed for updateUser by code and an upstream test. For linkIdentity it is confirmed by code reading only. Spike to close the gap (Supabase CLI and Docker, not run):
  1. `supabase init`; in config.toml set `[auth] enable_anonymous_sign_ins = true`, `enable_manual_linking = true`, `[auth.email] enable_confirmations = true`; optionally add `[auth.external.google]` credentials. Keys per https://github.com/supabase/cli/blob/bcda9df33f6e9235d68d6ce4178b02902403f1a5/apps/cli/src/shared/init/project-init.templates.ts#L159-L230.
  2. `supabase start`; the mail UI is on port 54324 (https://github.com/supabase/cli/blob/bcda9df33f6e9235d68d6ce4178b02902403f1a5/apps/cli/src/shared/init/project-init.templates.ts#L105-L114).
  3. With supabase-js: `signInAnonymously()`; print user id and decoded access-token claims.
  4. Insert a row keyed to that id into a test table with an FK to `auth.users(id)` and an owner RLS policy.
  5. `updateUser({email:'a@example.test', password:'Passw0rd!x'})`: expect no error, same id, still anonymous. A local stack has mailer validation off, so auth#2847 may reproduce only on a hosted project.
  6. Read the mail in the UI; `verifyOtp({email, token, type:'email_change'})`: expect same id, `is_anonymous` false, new session, step 4 row still readable.
  7. Compare claims of the old stored token and of `refreshSession()`: old true, refreshed false.
  8. New anonymous session: `updateUser({email:<existing user's email>})`: expect 422 `email_exists`.
  9. Optional Google: `linkIdentity({provider:'google'})`, check same id; repeat with an already linked Google account and read the redirect parameters.
  10. `select id, is_anonymous, email from auth.users; select user_id, provider from auth.identities;`
