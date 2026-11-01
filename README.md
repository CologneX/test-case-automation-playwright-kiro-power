# csv-test-automation (Kiro Power)

Turns a CSV of test cases (`test case id`, `test scenario`, `expected result`, `roles`) into plain Playwright Test (TypeScript) specs. AI is used to author and heal tests only. Test runs are plain `playwright test`, parallel, with no LLM calls.

```
csv-test-automation/
  POWER.md            entry point: onboarding, pipeline, steering map, precedence
                      (also holds the scaffold code, including the CSV/JSON/HTML results reporter)
  mcp.json            playwright-test MCP server, ported from init-agents
  README.md
  steering/
    normalize-csv.md  test-conventions.md  microsoft-auth.md  healing-rules.md   (custom)
    agent-planner.md  agent-generator.md   agent-healer.md                       (ported, Playwright 1.64.1)
    refresh-agents.md                                                            (custom)
```

## Install

1. Open the Kiro Powers panel (Ghosty icon with the lightning bolt).
2. Choose **Add Custom Power** → **Import power from a folder**. Older Kiro builds label this **Add power from Local Path**.
3. Select this `csv-test-automation` folder.
4. In a chat, mention "playwright" or "test case" to activate it, then ask Kiro to run onboarding.

Kiro registers the `playwright-test` MCP server for the Power itself. Your project's `.mcp.json` is not touched.

## Dry-run checklist (first 3 CSV rows)

Do this in the target project, not in this folder. "Rows 1-3" means the first three data rows below the header.

Setup
- [ ] Onboarding done: `bunx playwright test --list` shows `setup-<role>` and `<role>` for every role in the CSV.
- [ ] `.env` filled by you (BASE_URL + one username/password pair per role). Kiro never reads it.
- [ ] `microsoft-auth.md` first-run verification done: sign-in locators verified, signed-in landmark added to `tests/support/app.ts`.
- [ ] `bunx playwright test tests/seed.spec.ts` passes for each role used by rows 1-3.

Normalize ("normalize the first 3 rows of the CSV")
- [ ] `test-data/normalized/<TC-ID>.md` exists for each clear row, with numbered steps and `A<n>` assertions.
- [ ] Each assertion quotes a clause of the expected result. No clause is unmapped, and no assertion is invented.
- [ ] Unclear rows appear in `test-data/needs-clarification.md` with a question, and have no normalized file.

Plan ("plan the normalized rows")
- [ ] `specs/<TC-ID>.md` exists for each normalized row, starting with `<!-- status: draft -->` and a roles line.
- [ ] Plan steps match the normalized steps in order. Extra ideas sit under "Suggested extra coverage" only.
- [ ] You review each plan and change `draft` to `approved`.

Generate ("generate approved plans")
- [ ] One file per row: `tests/<csv-stem>/<tc-id>.spec.ts`.
- [ ] Title is `<TC-ID> <scenario>`. Tags are `@role-<slug>` for each row role. Import is from `../support/fixtures`.
- [ ] Locators are role/label/text based. No CSS/XPath without a `// why:` comment. No `waitForTimeout` or `networkidle`.
- [ ] Count of `expect` calls equals the count of `A<n>` assertions in the plan.

Run
- [ ] `bun run test` passes. On a second run within `AUTH_MAX_AGE_MINUTES`, the setup steps reuse the saved sessions and skip the Microsoft sign-in.
- [ ] `grep -rE "openai|anthropic|bedrock|mcp" tests/ playwright.config.ts` finds nothing.

Results
- [ ] Each run creates a new `reports/<YYYY-MM-DD_HH-mm-ss>/` folder with `results.csv`, `results.json`, `summary.html`, `playwright-report/`, and `test-results/`. Earlier run folders are unchanged.
- [ ] `reports/index.html` lists every run, newest first.
- [ ] `bunx playwright test --last-failed` re-runs only the previous run's failures.
- [ ] Rows 1-3 each appear once per role, with the expected result copied from the CSV.
- [ ] A row from `needs-clarification.md` shows `needs clarification`. A row with no spec shows `not automated`.
- [ ] For a failing test, "failed at" names the `A<n>` assertion, and "details" in the summary opens that test in the Playwright report.
- [ ] `grep -r "<a role password>" reports/` finds nothing.

Heal (break one locator on purpose, e.g. change a button name in a spec)
- [ ] Healer fixes the locator only, then passes with `--repeat-each=3`.
- [ ] Change an expected value in a spec: the healer asks you before changing it.
- [ ] `git diff` shows no edits to `specs/`, `test-data/normalized/`, or the CSV.

## Assumptions

- The CSV is UTF-8, comma-delimited, with a header row. A cell in `roles` can list several roles separated by `;`, `,`, `|`, or a newline. A row with several roles runs once per role.
- No test case CSV was found in this workspace, so the dry run above is written against "rows 1-3" of whichever CSV you point it at.
- Sign-in is Microsoft Entra ID through a browser redirect, with no 2FA and no Conditional Access prompts for the test accounts.
- The Microsoft email page locators were checked live on 2026-11-01. The password page, the "Use your password" button, and "Stay signed in?" could not be checked without a real account. They follow Microsoft's current wording and are flagged for verification on first run.
- The app's tokens are in cookies, localStorage, IndexedDB, or sessionStorage. All four are saved. Memory-only token caches still work, but each test does a silent redirect through Microsoft.
- Each run is saved to `reports/<YYYY-MM-DD_HH-mm-ss>/`, named with the machine's local time. Runs are kept until you delete them or set `REPORTS_KEEP`. Each run folder holds its own traces, so disk use grows with failures and retries.
- The `report` script uses `ls` and `$(...)`, so it needs a Unix shell (macOS, Linux, Git Bash).
- Chromium only, headless, for speed. Add browser projects if you need cross-browser coverage.
- Node 20.12+ (for `process.loadEnvFile`, so no dotenv dependency). The only added runtime dependency is `csv-parse` (MIT).
- The Power is in the legacy `POWER.md` + `steering/` format you asked for. Kiro's current docs recommend the Agent Plugins format (`plugin.json` + `skills/`) for new powers and state that `POWER.md` powers keep working. A legacy Power's MCP server is registered in the user-level `~/.kiro/settings/mcp.json` under the Powers section.
- Commands use bun/bunx. `mcp.json` keeps `npx` exactly as Playwright generated it. `npx` resolves the project-local Playwright, so the MCP server matches the project's version.

## What was removed or changed in the ported agents

Source: `npx playwright init-agents --loop=claude` with `@playwright/test` 1.64.1, run in `/tmp/pw-agents-port.XOjQ`, outside this workspace. Supported `--loop` values in 1.64.1: `claude`, `codex`, `copilot`, `opencode`, `vscode`, `vscode-legacy`. Generated files: three agent definitions in `.claude/agents/`, `.mcp.json`, `seed.spec.ts`, and `specs/README.md`.

| Item | Change | Why |
|---|---|---|
| Frontmatter `name`, `model: sonnet`, `color` | Removed | Claude Code subagent settings. Kiro does not use them. |
| Frontmatter `description` | Moved to "When to use (original description)", text unchanged | Kept the wording, including the generator's input example. |
| Frontmatter `tools:` | Moved to "MCP tools this agent uses". Removed the `mcp__playwright-test__` prefix and the Claude built-ins (`Glob`, `Grep`, `Read`, `LS`, `Edit`, `MultiEdit`, `Write`) | The prefix is Claude's naming scheme and the built-ins are Claude permissions. The MCP tool names themselves are unchanged and match the server's `tools/list`. |
| Agent bodies | Kept verbatim under "Original instructions". Checked with `diff`: identical except the trailing newline. | You asked to keep the original wording. |
| Added "Power adaptation" sections | New text above each verbatim section | Planner reads normalized CSV rows and still explores the live app to verify. Generator only processes approved plans and applies the conventions. Healer overrides are listed in a table. |
| Healer lines "Fixing assertions and expected values" and "Do not ask user questions..." | Kept in the text, overridden by `healing-rules.md` | They conflict with your heal rule. The precedence rule in POWER.md makes your rule win. |
| Healer "continue until the test runs successfully" | Overridden: max 3 attempts | Prevents a loop from weakening a test until it passes. |
| `.mcp.json` | Copied as-is to the Power's `mcp.json` (`playwright-test`: `npx playwright run-test-mcp-server`) | It is the only server the agents use. It includes the browser tools, so no separate browser MCP server was added. Nothing was written to your project's `.mcp.json`. |
| `seed.spec.ts` | Same `describe`/`test('seed')` shape and `// generate code here.` marker. Added `@seed` tag, fixtures import, `goto('/')`, `expectSignedIn` | It must start signed in through the role project's storageState. The tag lets role projects include the seed. `bun run test` excludes it. |
| `specs/README.md` | Not ported. Onboarding creates an empty `specs/` folder. | Placeholder text only. |
