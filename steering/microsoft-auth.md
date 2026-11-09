# Microsoft sign-in

Code in the project (from `onboarding.md`): `tests/auth.setup.ts`, `tests/support/{roles,app,fixtures}.ts`.

## Model

- Per role: `setup-<slug>` runs `auth.setup.ts` and saves `.auth/<slug>.json` (cookies, localStorage, IndexedDB) and `.auth/<slug>.session.json` (sessionStorage). `<slug>` depends on it, loads that state, and runs tests tagged `@role-<slug>`.
- Credentials: `.env` `ROLE_<SLUG>_USERNAME` / `ROLE_<SLUG>_PASSWORD`. Never ask for, read, print, log, or commit them. Test accounts only.

## First-run verification (once per project)

1. Ask the user to confirm `.env` is filled.
2. `bunx playwright test --project=setup-<one-role> --headed`.
3. A sign-in step fails: walk the Microsoft pages from `BASE_URL` with `browser_*` tools and replace only the failing locator in `signInWithMicrosoft` (`browser_generate_locator`). Update its "verified" comment (Playwright version, date).
4. Add ONE stable signed-in landmark (user menu, account button) to `expectSignedIn` in `tests/support/app.ts`. Without it the URL check can pass before the redirect to Microsoft.
5. `bunx playwright test tests/seed.spec.ts` passes for every role.

## Sessions

- Setup reuses a session younger than `AUTH_MAX_AGE_MINUTES` (default 45) that still passes `expectSignedIn`; otherwise it signs in again. Keep it below the app's real session lifetime.
- Force fresh sign-in: delete `.auth/<slug>*.json` or run with `AUTH_MAX_AGE_MINUTES=0`.
- Mid-run redirect to `login.microsoftonline.com`: session expired. Lower `AUTH_MAX_AGE_MINUTES`; never add sign-in code to tests.
- "Stay signed in?" is answered Yes by the setup, which races it against the app URL. No fixed waits.
- Tokens not in sessionStorage (memory only): report it. Each test then redirects silently through Microsoft (slower, still works).

## New role in the CSV

Add an empty `ROLE_<SLUG>_*` pair to `.env.example`, ask the user to fill `.env`, then `bunx playwright test --list` shows `setup-<slug>` and `<slug>`.

## Never

Script or bypass 2FA, CAPTCHA, or security questions. If Microsoft shows one, stop: the test account needs it disabled.
