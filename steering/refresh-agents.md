# Refresh the ported agents after a Playwright upgrade

Run this when the project's `@playwright/test` version changes. Never run `init-agents` in the real project: it overwrites `.mcp.json` and writes `.claude/`, `specs/README.md`, and `seed.spec.ts`.

## 1. Check versions

```bash
bunx playwright --version              # project version
npm view @playwright/test version      # latest published
```

Read https://playwright.dev/docs/test-agents and the release notes for agent or MCP changes.

## 2. Generate in a throwaway directory outside the workspace

```bash
TMP=$(mktemp -d /tmp/pw-agents-port.XXXX) && cd "$TMP"
printf '{"name":"pw-agents-port","private":true}\n' > package.json
bun add -d @playwright/test@<project version>
bunx playwright init-agents --help     # confirm --loop choices
bunx playwright init-agents --loop=claude    # use --loop=opencode if claude is not offered
```

## 3. Read every generated file

- `.claude/agents/playwright-test-planner.md`
- `.claude/agents/playwright-test-generator.md`
- `.claude/agents/playwright-test-healer.md`
- `.mcp.json`
- `seed.spec.ts`
- `specs/README.md` and any other new file (`find . -path ./node_modules -prune -o -type f -print`)

## 4. Diff against the current port

For each agent, compare the generated body with the "Original instructions (verbatim)" section of `steering/agent-<name>.md`, and the frontmatter `tools:` list with "MCP tools this agent uses" (strip the `mcp__playwright-test__` prefix first). Compare `.mcp.json` with this Power's `mcp.json`.

```bash
for a in planner generator healer; do
  echo "== $a"
  diff <(sed '1,/^---$/d' "$TMP/.claude/agents/playwright-test-$a.md" | sed '/./,$!d') \
       <(sed -n '/^## Original instructions/,$p' "<power>/steering/agent-$a.md" | tail -n +3)
done
```

A lone "No newline at end of file" difference is expected. Anything else is a real change.

Optional: list the server's tools to catch renamed tools.

```bash
printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' '{"jsonrpc":"2.0","method":"notifications/initialized"}' '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  | (bunx playwright run-test-mcp-server & sleep 6; kill $!) 2>/dev/null | tail -1
```

## 5. Apply changes by hand

- Show the user the diff summary before editing. Never overwrite a ported file wholesale.
- Update only the verbatim section, the tools list, and the header (version, date).
- Re-check every "Power adaptation" override still points at text that exists. Update the override table in `agent-healer.md` if wording changed.
- Copy into `mcp.json` only servers the agents' tool lists depend on. Do not add unrelated entries. Do not touch the project's `.mcp.json`.
- Compare the new `seed.spec.ts` with the project seed. Keep the project seed authenticated (role project + `expectSignedIn`).

## 6. Verify and clean up

- `bunx playwright test --list` in the project still shows setup and role projects.
- Plan and generate one known row as a smoke check.
- `rm -rf "$TMP"`
