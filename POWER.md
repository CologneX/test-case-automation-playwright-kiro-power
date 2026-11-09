---
name: "csv-test-automation"
displayName: "Test Automation"
description: "Turn a CSV of test cases (test case id, test scenario, expected result, roles) into deterministic, parallel Playwright Test code for a Microsoft sign-in web app. AI is used only to author and heal tests, never during a test run."
keywords: ["test case", "CSV scenarios", "playwright", "automation", "test plan", "playwright healer", "csv test cases", "e2e automation"]
author: "Kyrell (Axrail)"
---

# CSV Test Automation

Each CSV row (`test case id`, `test scenario`, `expected result`, `roles`) becomes one plain Playwright Test spec. Runs use `playwright test` with no AI. AI only plans, generates, and heals.

## Rules

- No LLM, AI SDK, or MCP calls in test code or the Playwright config.
- One CSV row = one test = one spec file.
- Roles come from the CSV `roles` column. Never hardcode them.
- Selectors come from the live app via the `playwright-test` MCP server. Never invent one.
- Credentials, tokens, and storageState stay in the project (`.env`, `.auth/`, gitignored). Never in this Power, never read or printed.
- Never write the project's `.mcp.json`.
- Conflicts: `test-conventions.md` and the rules in `agent-healer.md` win over everything else.

## Pipeline

Plan, Generate, and Heal run as up to 5 parallel sub-agents. Main agent: read `parallel-agents.md` first.

1. **Plan** (`agent-planner.md`), no browser: CSV rows become `specs/<TC-ID>.md` (`status: draft`) with numbered steps and `A<n>` assertions. Unclear rows go to `test-data/needs-clarification.md`. Stop for user review.
2. **Generate** (`agent-generator.md` + `test-conventions.md`): approved plans only. The only live check. Writes `tests/<csv-stem>/<tc-id>.spec.ts`.
3. **Run**: `bun run test`. Parallel, one sign-in per role, results in `reports/<YYYY-MM-DD_HH-mm-ss>/`.
4. **Heal** (`agent-healer.md` + `test-conventions.md`): locators and waits only. Ask before touching an assertion. App broken: `test.fixme()` and report.

## Read only when needed

| Situation | File |
|---|---|
| First use, or a scaffold file is missing | `onboarding.md`, then `microsoft-auth.md` |
| Sign-in fails, session expired, new role | `microsoft-auth.md` |
| User asks for results | `results.md` |
| Playwright upgraded | `refresh-agents.md` |

## MCP server `playwright-test`

`bunx playwright run-test-mcp-server`. Needs `@playwright/test` in the project and `playwright.config.ts` in the workspace root (else add `"-c", "<subfolder>"` to `args`). Add `"--headless"` to `args` for faster authoring.

## Running tests (no AI)

```bash
bun run test                                   # all roles, parallel
bunx playwright test --project=<role-slug>     # one role
bunx playwright test --grep "TC-001"           # one test case
bunx playwright test --last-failed             # re-run failures
```
