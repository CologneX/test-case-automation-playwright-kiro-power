# Refresh the ported agents after a Playwright upgrade

Run when the project's `@playwright/test` version changes. Never run `init-agents` in the real project: it overwrites `.mcp.json` and writes `.claude/`, `specs/README.md`, `seed.spec.ts`.

1. Versions: `bunx playwright --version` (project), `npm view @playwright/test version` (latest). Read https://playwright.dev/docs/test-agents and the release notes for agent or MCP changes.

2. Generate in a throwaway directory:

```bash
TMP=$(mktemp -d /tmp/pw-agents-port.XXXX) && cd "$TMP"
printf '{"name":"pw-agents-port","private":true}\n' > package.json
bun add -d @playwright/test@<project version>
bunx playwright init-agents --loop=claude    # --loop=opencode if claude is not offered
```

3. Diff each generated body against the matching "Original instructions" in `steering/ported-originals.md`, and its frontmatter `description` and `tools:` (strip `mcp__playwright-test__`) against that agent's other sections. Compare `.mcp.json` with this Power's `mcp.json`, and `seed.spec.ts` with the project seed. Check for new files: `find . -path ./node_modules -prune -o -type f -print`.

Optional, list the server's tools to catch renames:

```bash
printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' '{"jsonrpc":"2.0","method":"notifications/initialized"}' '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  | (bunx playwright run-test-mcp-server & sleep 6; kill $!) 2>/dev/null | tail -1
```

4. Show the user a diff summary before editing. Then:
   - Replace the changed sections of `ported-originals.md` and update its version and date.
   - Carry real behavior changes (new steps, renamed tools, new rules) into the compact `agent-planner.md`, `agent-generator.md`, `agent-healer.md`. Keep the Power rules (no assertion changes without asking, 3 attempts, no browser in Plan, sub-agent return format).
   - Copy into `mcp.json` only servers the agents need. Never touch the project's `.mcp.json`.
   - Keep the project seed authenticated (role project + `expectSignedIn`).

5. Verify: `bunx playwright test --list` still shows setup and role projects; plan and generate one known row; `rm -rf "$TMP"`.
