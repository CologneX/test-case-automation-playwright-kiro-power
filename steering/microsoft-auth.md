# Microsoft sign-in

The code lives in the project, scaffolded from POWER.md Onboarding: `tests/auth.setup.ts`, `tests/support/roles.ts`, `tests/support/app.ts`, `tests/support/fixtures.ts`.

## Model

- One setup project per role (`setup-<role-slug>`) runs `auth.setup.ts` once and saves `.auth/<role-slug>.json` (cookies, localStorage, IndexedDB) and `.auth/<role-slug>.session.json` (sessionStorage).
- One test project per role (`<role-slug>`) depends on its setup project, loads that storageState, and runs only tests tagged `@role-<role-slug>`.
- Roles come from the CSV `roles` column at config load. Never hardcode a role.
- Credentials come from `.env`: `ROLE_<SLUG>_USERNAME`, `ROLE_<SLUG>_PASSWORD`. Never ask for, print, log, or commit them.
- `.auth/` and `.env` are gitignored. Never copy them into the Power folder or into reports.

## First-run verification (once per project)

1. Ask the user to confirm `.env` is filled. Do not read it.
2. Run `bunx playwright test --project=setup-<one-role> --headed`.
3. If a sign-in step fails, use the MCP `browser_*` tools to open `BASE_URL` and walk the Microsoft pages. Replace only the failing locator in `signInWithMicrosoft` with the one you discover (`browser_generate_locator`). Update the "verified" comment with the Playwright version and date.
4. Once signed in, discover ONE stable signed-in landmark (user menu, account button, app shell heading) and add it to `expectSignedIn` in `tests/support/app.ts`. Without it, the URL check alone can pass before the app redirects to Microsoft.
5. Run every setup project: `bunx playwright test tests/seed.spec.ts`. All must pass.

## "Stay signed in?" prompt

- Microsoft may show it after the password. Click `Yes`. That sets the persistent Microsoft cookie, so silent re-auth works and sessions last longer.
- The setup races the app URL against the prompt, so it works whether or not the prompt appears. Do not add fixed waits.

## Session expiry and refresh

- Setup reuses a saved session if it is younger than `AUTH_MAX_AGE_MINUTES` (default 45) and `expectSignedIn` passes in a fresh context. Otherwise it signs in again.
- Set `AUTH_MAX_AGE_MINUTES` below the app's real token or session lifetime.
- Force a fresh sign-in: delete `.auth/<role-slug>*.json`, or run with `AUTH_MAX_AGE_MINUTES=0`.
- If tests fail with a redirect to `login.microsoftonline.com` mid-run, the session expired during the run. Lower `AUTH_MAX_AGE_MINUTES`. Do not add sign-in code to tests.
- If the app keeps tokens in sessionStorage (MSAL default), `fixtures.ts` restores them. If tokens live elsewhere (memory only), report it: each test will do a silent redirect through Microsoft, which is slower but still works.

## New role in the CSV

1. Add the `ROLE_<SLUG>_*` pair to `.env.example` (empty values).
2. Ask the user to fill `.env`.
3. Run `bunx playwright test --list` to confirm the new `setup-<slug>` and `<slug>` projects.

## Never

- Never enable or script 2FA, CAPTCHA, or security-question bypasses. If Microsoft shows one, stop and tell the user the test account needs it disabled.
- Never sign in with real user accounts. Test accounts only.
