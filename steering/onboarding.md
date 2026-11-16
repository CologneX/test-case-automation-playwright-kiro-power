# Onboarding (first use in a project)

Read only on first use, or when a scaffold file is missing. Copy code blocks exactly. Do not ask for, read, or store real credentials. The user fills `.env`.

## Step 1: Check tools

- Run `node -v`. Require Node 20.12 or later (for `process.loadEnvFile`). If missing, stop and tell the user to install Node LTS.
- Package manager: use bun if `bun -v` works (`bun add -d`, `bunx`), otherwise npm (`npm i -D`, `npx`). The commands below show bun; swap in npm/npx if needed.
- Run `bunx playwright --version`. If `@playwright/test` is not a dev dependency, continue to Step 2.

## Step 2: Install dependencies

```bash
bun add -d @playwright/test csv-parse @types/node @aws/bedrock-token-generator
bunx playwright install chromium
```

`@aws/bedrock-token-generator` is used only by the optional post-run describe step (`describe.md`). Tests never import it.

## Step 3: Locate the CSV

Ask the user for the CSV path if it is not at `test-data/test-cases.csv`. Put its path in `.env.example` as `TEST_CASES_CSV`. Check the header has the columns `test case id`, `test scenario`, `expected result`, `roles` (case-insensitive). If not, stop and report the actual headers.

## Step 4: Scaffold missing files

Create only files that do not exist. If a file exists, show the user a diff of what you would add and ask first.

`playwright.config.ts`:

```ts
import fs from 'node:fs';
import path from 'node:path';
import { defineConfig, devices } from '@playwright/test';
import { authFile, readRoles, roleTag } from './tests/support/roles';
import { escapeRegExp } from './tests/support/app';
import { REPORTS_DIR, runDir } from './tests/support/run';

if (fs.existsSync('.env')) process.loadEnvFile('.env');
if (!process.env.BASE_URL) throw new Error('Set BASE_URL in .env (copy .env.example)');

const roles = readRoles();
// Every run gets its own folder: reports/<YYYY-MM-DD_HH-mm-ss>/
const run = runDir();
// Keep --last-failed working across run folders.
process.env.PLAYWRIGHT_LAST_RUN_OUTPUT_FILE ??= path.join(REPORTS_DIR, '.last-run.json');

export default defineConfig({
  testDir: './tests',
  outputDir: path.join(run, 'test-results'),
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 1 : 0,
  workers: process.env.PW_WORKERS ?? undefined,
  // --list runs nothing, so it gets no run folder.
  reporter: process.argv.includes('--list') ? [['list']] : [
    ['list'],
    ['html', { open: 'never', outputFolder: path.join(run, 'playwright-report') }],
    // Per-CSV-row results, no AI: <run>/results.csv, results.json, summary.html, plus reports/index.html
    ['./tests/support/results-reporter.ts', { outputDir: run, htmlReportDir: path.join(run, 'playwright-report') }],
  ],
  use: {
    baseURL: process.env.BASE_URL,
    locale: 'en-US',
    trace: 'on',
    screenshot: 'on',
    video: 'on',
  },
  projects: [
    // One setup project per role: signs in once and saves .auth/<role>.json.
    ...roles.map((role) => ({
      name: `setup-${role.slug}`,
      testMatch: /auth\.setup\.ts$/,
      metadata: { role },
      use: { ...devices['Desktop Chrome'], trace: 'off' as const, screenshot: 'off' as const, video: 'off' as const },
    })),
    // One test project per role: reuses the saved session and runs only tests tagged with that role.
    ...roles.map((role) => ({
      name: role.slug,
      dependencies: [`setup-${role.slug}`],
      grep: [new RegExp(`${escapeRegExp(roleTag(role.slug))}(?![\\w-])`), /@seed(?![\w-])/],
      use: { ...devices['Desktop Chrome'], storageState: authFile(role.slug) },
    })),
  ],
});
```

`tests/support/roles.ts`:

```ts
import fs from 'node:fs';
import path from 'node:path';
import { parse } from 'csv-parse/sync';

// Roles come from the CSV "roles" column. Never hardcode role names.
export type Role = { name: string; slug: string };
export type CsvRow = Record<string, string>;

// Read lazily: playwright.config.ts loads .env after its imports run.
export const csvPath = () => process.env.TEST_CASES_CSV ?? 'test-data/test-cases.csv';
const ROLE_SEPARATOR = /[;,|\n]/;

export function slugify(value: string): string {
  return value.trim().toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '');
}

export function readCsvRows(file: string = csvPath()): CsvRow[] {
  return parse(fs.readFileSync(file), {
    columns: (header: string[]) => header.map((h) => h.trim().toLowerCase()),
    bom: true,
    skip_empty_lines: true,
    trim: true,
  }) as CsvRow[];
}

export function splitRoles(cell: string = ''): Role[] {
  return cell
    .split(ROLE_SEPARATOR)
    .map((name) => ({ name: name.trim(), slug: slugify(name) }))
    .filter((role) => role.slug);
}

export function readRoles(file: string = csvPath()): Role[] {
  const roles = new Map<string, Role>();
  for (const row of readCsvRows(file)) {
    for (const role of splitRoles(row['roles'])) if (!roles.has(role.slug)) roles.set(role.slug, role);
  }
  if (roles.size === 0) throw new Error(`No roles found in the "roles" column of ${file}`);
  return [...roles.values()];
}

export const roleTag = (slug: string) => `@role-${slug}`;
export const authFile = (slug: string) => path.join('.auth', `${slug}.json`);
export const sessionFile = (slug: string) => path.join('.auth', `${slug}.session.json`);

export function credentialsFor(role: Role) {
  const key = role.slug.toUpperCase().replace(/-/g, '_');
  const username = process.env[`ROLE_${key}_USERNAME`];
  const password = process.env[`ROLE_${key}_PASSWORD`];
  if (!username || !password) {
    throw new Error(`Set ROLE_${key}_USERNAME and ROLE_${key}_PASSWORD in .env for role "${role.name}"`);
  }
  return { username, password };
}
```

`tests/support/app.ts`:

```ts
import { expect, type Page } from '@playwright/test';

export const escapeRegExp = (s: string) => s.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
export const appOrigin = () => new URL(process.env.BASE_URL!).origin;
export const appUrlPattern = () => new RegExp(`^${escapeRegExp(appOrigin())}`);

// Passes only when the page is the app, signed in.
export async function expectSignedIn(page: Page) {
  await expect(page).toHaveURL(appUrlPattern());
  // ONBOARDING: add ONE signed-in landmark discovered on the live app via MCP
  // (browser_snapshot + browser_generate_locator), for example the user menu:
  // await expect(page.getByRole('button', { name: '<discovered name>' })).toBeVisible();
}
```

`tests/support/fixtures.ts` (every spec imports `test` and `expect` from here):

```ts
import fs from 'node:fs';
import { test as base, expect } from '@playwright/test';
import { sessionFile } from './roles';
import { appOrigin } from './app';

// Every spec imports { test, expect } from this file.
// Restores the app's sessionStorage (MSAL default cache) saved by the setup project.
export const test = base.extend<{ restoreSessionStorage: void }>({
  restoreSessionStorage: [
    async ({ context }, use, testInfo) => {
      const file = sessionFile(testInfo.project.name);
      if (fs.existsSync(file)) {
        await context.addInitScript(
          ([origin, json]) => {
            if (window.location.origin !== origin) return;
            for (const [key, value] of Object.entries(JSON.parse(json) as Record<string, string>)) {
              if (window.sessionStorage.getItem(key) === null) window.sessionStorage.setItem(key, value);
            }
          },
          [appOrigin(), fs.readFileSync(file, 'utf8')] as const,
        );
      }
      await use();
    },
    { auto: true },
  ],
});

export { expect };
```

`tests/support/run.ts` (one timestamped folder per run):

```ts
import fs from 'node:fs';
import path from 'node:path';

export const REPORTS_DIR = 'reports';

// One folder per run: reports/<YYYY-MM-DD_HH-mm-ss> (local time).
// Playwright loads the config in the main process and again in every worker.
// The main process picks a new id on every load (also for long-lived MCP runs);
// workers (TEST_WORKER_INDEX is set) reuse it through PW_RUN_ID.
export function runDir(): string {
  if (process.env.TEST_WORKER_INDEX === undefined || !process.env.PW_RUN_ID) {
    const d = new Date();
    const p = (n: number) => String(n).padStart(2, '0');
    const stamp = `${d.getFullYear()}-${p(d.getMonth() + 1)}-${p(d.getDate())}_${p(d.getHours())}-${p(d.getMinutes())}-${p(d.getSeconds())}`;
    let id = stamp;
    for (let i = 2; fs.existsSync(path.join(REPORTS_DIR, id)); i++) id = `${stamp}-${i}`;
    process.env.PW_RUN_ID = id;
  }
  return path.join(REPORTS_DIR, process.env.PW_RUN_ID);
}
```

`tests/support/error-text.ts` (turns Playwright errors into one plain sentence for the report; rule-based, no AI):
```ts
// Turns Playwright error messages into one plain sentence for the report, for example
//   "Button matching /^Edit/ not found – could not click it (waited 10 s)".
// Only what the page shows or lacks is stated; nothing here knows what it means for the product.
// Unknown message shapes fall back to the first lines of the original text (see describeErrors).

const ANSI = /\u001b\[[0-9;]*m/g;

type Call = { name: string; args: string };

// Splits "getByRole('cell', {...}).locator('x').first()" into its calls, honouring quotes and nested parentheses.
function parseChain(expr: string): Call[] | null {
  const calls: Call[] = [];
  let i = 0;
  while (i < expr.length) {
    const m = /^\.?([A-Za-z]+)\(/.exec(expr.slice(i));
    if (!m) return null;
    const start = i + m[0].length;
    let depth = 1;
    let quote: string | null = null;
    let j = start;
    for (; j < expr.length && depth > 0; j++) {
      const c = expr[j];
      if (quote) {
        if (c === '\\') j++;
        else if (c === quote) quote = null;
      } else if (c === "'" || c === '"' || c === '`') quote = c;
      else if (c === '(') depth++;
      else if (c === ')') depth--;
    }
    if (depth !== 0) return null;
    calls.push({ name: m[1], args: expr.slice(start, j - 1) });
    i = j;
  }
  return calls.length ? calls : null;
}

const ROLE_WORDS: Record<string, string> = {
  textbox: 'text box',
  searchbox: 'search box',
  combobox: 'dropdown',
  columnheader: 'column header',
  rowheader: 'row header',
  listbox: 'list',
  menuitem: 'menu item',
  spinbutton: 'number field',
};

const STRING_OR_REGEX = String.raw`(?:'((?:[^'\\]|\\.)*)'|"((?:[^"\\]|\\.)*)"|(\/(?:[^\/\\]|\\.)+\/[a-z]*))`;

function firstArg(args: string): string | null {
  const m = new RegExp(`^\\s*${STRING_OR_REGEX}`).exec(args);
  return m ? (m[1] ?? m[2] ?? m[3] ?? null) : null;
}

function quoted(args: string): string {
  const m = new RegExp(`^\\s*${STRING_OR_REGEX}`).exec(args);
  if (!m) return '';
  return m[3] ? `matching ${shorten(m[3])}` : `"${shorten(m[1] ?? m[2] ?? '')}"`;
}

function shorten(text: string, max = 70): string {
  const t = text.replace(/\\(.)/g, '$1');
  return t.length > max ? `${t.slice(0, max - 1)}…` : t;
}

// "getByRole('button', { name: 'Edit documentation' })" -> 'button "Edit documentation"'
export function describeLocator(expr: string): string {
  const calls = parseChain(expr.trim());
  if (!calls) return shorten(expr.trim(), 90);
  const parts: string[] = [];
  for (const call of calls) {
    const { name, args } = call;
    if (name === 'getByRole') {
      const role = firstArg(args) ?? 'element';
      const word = ROLE_WORDS[role] ?? role;
      const nameMatch = new RegExp(`name:\\s*${STRING_OR_REGEX}`).exec(args);
      const label = nameMatch ? (nameMatch[3] ? `matching ${shorten(nameMatch[3])}` : `"${shorten(nameMatch[1] ?? nameMatch[2] ?? '')}"`) : '';
      parts.push(label ? `${word} ${label}` : word);
    } else if (name === 'getByText') parts.push(`text ${quoted(args)}`);
    else if (name === 'getByLabel') parts.push(`field labelled ${quoted(args)}`);
    else if (name === 'getByPlaceholder') parts.push(`field with placeholder ${quoted(args)}`);
    else if (name === 'getByTestId') parts.push(`element with test id ${quoted(args)}`);
    else if (name === 'getByAltText' || name === 'getByTitle') parts.push(`element ${quoted(args)}`);
    else if (name === 'locator') {
      const sel = firstArg(args) ?? '';
      if (/following-sibling::(td|\*)\[1\]/.test(sel)) parts.push('next cell');
      else parts.push(`element ${shorten(sel, 50)}`);
    } else if (name === 'nth') parts.push(`item ${args.trim()}`);
    // filter(...), first() and last() change which element is chosen but add nothing worth reading.
  }
  // A bare "table" container only adds noise when something more specific is inside it.
  if (parts.length > 1 && parts[0] === 'table') parts.shift();
  if (!parts.length) return shorten(expr.trim(), 90);
  // Merge "<x>" + "next cell" into "cell next to <x>".
  const merged: string[] = [];
  for (const part of parts) {
    if (part === 'next cell' && merged.length) merged.push(`cell next to ${merged.pop()}`);
    else merged.push(part);
  }
  return merged.reverse().join(' in ');
}

const capital = (s: string) => (s ? s[0].toUpperCase() + s.slice(1) : s);
const seconds = (ms: string | number) => `${Math.round(Number(ms) / 100) / 10} s`.replace(/\.0 s$/, ' s');

const ACTION_TEXT: Record<string, string> = {
  click: 'click it',
  dblclick: 'double-click it',
  fill: 'type into it',
  type: 'type into it',
  pressSequentially: 'type into it',
  press: 'press a key on it',
  check: 'tick it',
  uncheck: 'untick it',
  setChecked: 'tick it',
  selectOption: 'pick an option in it',
  hover: 'hover over it',
  focus: 'focus it',
  clear: 'clear it',
  setInputFiles: 'upload a file to it',
  dispatchEvent: 'use it',
  scrollIntoViewIfNeeded: 'scroll to it',
  textContent: 'read it',
  innerText: 'read it',
  inputValue: 'read it',
  getAttribute: 'read it',
  ariaSnapshot: 'read it',
};

function actionError(message: string): string | null {
  const head = /(locator|page)\.(\w+): (?:Test timeout of (\d+)ms exceeded|Timeout (\d+)ms exceeded)/.exec(message);
  if (!head) return null;
  const [, target, action, testLimit, explicit] = head;
  const limit = testLimit
    ? `test time limit of ${seconds(testLimit)} ran out`
    : `waited ${seconds(explicit)}`;

  if (target === 'page' && action === 'goto') {
    const url = /navigating to "([^"]+)"/.exec(message)?.[1] ?? '';
    return `Page did not finish loading${url ? `: ${url}` : ''} (${limit})`;
  }
  if (target === 'page' && action === 'waitForURL') return `Page did not reach the expected address (${limit})`;

  const waiting = /^\s*-\s*waiting for (?!navigation)(.+)$/m.exec(message)?.[1]?.trim();
  if (!waiting) return null;
  // Playwright prints "waiting for locator(...)" or "waiting for getByRole(...)".
  const desc = describeLocator(waiting.replace(/^locator\(['"]?(.*?)['"]?\)$/, '$1'));

  if (action === 'waitFor') return `${capital(desc)} did not appear (${limit})`;

  const found = /locator resolved to/.test(message);
  if (found) {
    const reasons = [...message.matchAll(/^\s*-\s*element (is not (?:visible|enabled|editable|stable|attached)|intercepts pointer events|is outside of the viewport)/gm)];
    const reason = reasons.length ? reasons[reasons.length - 1][1] : '';
    const why = reason
      ? `found but ${reason.replace(/^is /, '').replace('intercepts pointer events', 'blocked by another element')}`
      : 'found but did not become usable';
    return `${capital(desc)} ${why} – could not ${ACTION_TEXT[action] ?? 'use it'} (${limit})`;
  }
  return `${capital(desc)} not found – could not ${ACTION_TEXT[action] ?? `run ${action} on it`} (${limit})`;
}

function expectError(message: string): string | null {
  const header = /expect\(\w+\)\.(not\.)?(\w+)\(/.exec(message);
  const locatorLine = /^\s*Locator:\s*(.+)$/m.exec(message)?.[1];
  const expected = /^\s*Expected(?: string| pattern| value)?:\s*(.+)$/m.exec(message)?.[1]?.trim();
  const received = /^\s*Received(?: string| value| pattern)?:\s*(.+)$/m.exec(message)?.[1]?.trim();
  const timeout = /^\s*Timeout:\s*(\d+)ms/m.exec(message)?.[1];
  const wait = timeout ? ` (waited ${seconds(timeout)})` : '';

  // A message written into the test, expect(locator, 'message'), becomes the first line of the error, above
  // Playwright's own "expect(locator).toX() failed" line. It states the meaning, so it is shown as written;
  // expected/got is appended only when the page showed a different value (not when the element is missing).
  const firstLine = message.split('\n')[0].replace(/^Error:\s*/, '').trim();
  const custom = firstLine && !/^expect\(/.test(firstLine) && !/^(locator|page)\.\w+:/.test(firstLine) ? firstLine : '';
  if (custom && (header || locatorLine)) {
    const missing = /element\(s\) not found/.test(message);
    const hasValues = expected !== undefined && received !== undefined && !missing;
    return hasValues ? `${custom} (expected ${expected}, got ${received})` : custom;
  }
  if (!header) return null;
  const negated = !!header[1];
  const matcher = header[2];

  if (/strict mode violation/.test(message)) {
    const n = /resolved to (\d+) elements/.exec(message)?.[1];
    const expr = /strict mode violation: (.+?) resolved to/s.exec(message)?.[1]?.replace(/\s+/g, ' ');
    return `${capital(describeLocator(expr ?? locatorLine ?? ''))} matches ${n ?? 'several'} elements, expected exactly one`;
  }

  // toHaveURL / toHaveTitle have no locator.
  if (!locatorLine) {
    if (matcher === 'toHaveURL') return `Page address is ${received ?? 'different'}, expected ${expected ?? 'another address'}${wait}`;
    if (matcher === 'toHaveTitle') return `Page title is ${received ?? 'different'}, expected ${expected ?? 'another title'}${wait}`;
    return null;
  }

  const desc = capital(describeLocator(locatorLine));
  if (/element\(s\) not found/.test(message) && !negated) return `${desc} not found${wait}`;
  if (matcher === 'toBeVisible') {
    return negated ? `${desc} is still visible${wait}` : `${desc} is not visible${wait}`;
  }
  if (matcher === 'toHaveCount') {
    const want = expected ?? '?';
    const got = received ?? '?';
    return want === '0' ? `${desc}: expected none, found ${got}${wait}` : `${desc}: expected ${want}, found ${got}${wait}`;
  }
  if (expected !== undefined && received !== undefined) {
    return `${desc}: expected ${negated ? 'not ' : ''}${expected}, got ${received}${wait}`;
  }
  return `${desc}: ${matcher} check failed${wait}`;
}

function otherError(message: string): string | null {
  const net = /net::(ERR_[A-Z_]+)/.exec(message)?.[1];
  if (net) return `Network error while loading the page (${net})`;
  if (/Target (page, context or browser|closed)|has been closed/.test(message)) return 'The browser or page was closed during the test';
  return null;
}

function genericTimeout(message: string): string | null {
  const m = /^\s*(?:Error: )?Test timeout of (\d+)ms exceeded\.?\s*$/m.exec(message);
  return m ? `Test ran out of time (limit ${seconds(m[1])})` : null;
}

const firstLines = (message: string) =>
  message.split('\n').map((l) => l.trim()).filter(Boolean).slice(0, 3).join(' | ').slice(0, 500);

// messages: every error of the last attempt, in order. Returns one sentence per distinct error, joined with "; ".
export function describeErrors(messages: string[]): string {
  const cleaned = messages.map((m) => m.replace(ANSI, ''));
  const sentences: string[] = [];
  let timeoutOnly: string | null = null;
  for (const message of cleaned) {
    const specific = actionError(message) ?? expectError(message) ?? otherError(message);
    if (specific) {
      if (!sentences.includes(specific)) sentences.push(specific);
      continue;
    }
    const plain = genericTimeout(message);
    if (plain) timeoutOnly = plain;
    else {
      const raw = firstLines(message);
      if (raw && !sentences.includes(raw)) sentences.push(raw);
    }
  }
  // "Test timeout" is the symptom when a later error already names the missing element.
  if (!sentences.length && timeoutOnly) sentences.push(timeoutOnly);
  return sentences.slice(0, 3).join('; ');
}
```
`tests/support/error-cell.ts` (renders the error cell of a failed row in `summary.html`; shared by the reporter and `describe-failures.ts`):
```ts
// The "error" cell of a failed row in summary.html, shared by the reporter (rule-based text) and by
// describe-failures.ts (which swaps in the AI description). Markers let the describe step replace exactly one cell.
//
//   visible:    AI description when there is one, otherwise the rule-based sentence (error-text.ts)
//   collapsed:  "technical error" = the rule-based sentence (when the AI text is shown) and the original Playwright error

export type ErrorCellRow = {
  error: string;
  technicalError?: string;
  aiDescription?: string;
  aiModel?: string;
};

const esc = (v: string) => v.replace(/[&<>"']/g, (c) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' })[c]!);

export function renderErrorCell(row: ErrorCellRow, index: number): string {
  const ai = row.aiDescription?.trim();
  let main = '';
  if (ai) {
    main = `<div class="ai-description"><b>AI description</b> <small>(${esc(row.aiModel ?? 'AI')}, may be inaccurate)</small><br>${esc(ai)}</div>`;
  } else if (row.error) {
    main = `<code>${esc(row.error)}</code>`;
  }
  const technical: string[] = [];
  if (ai && row.error) technical.push(`<div><b>Summary</b> <code>${esc(row.error)}</code></div>`);
  if (row.technicalError) technical.push(`<div><b>Playwright error</b> <code>${esc(row.technicalError)}</code></div>`);
  const details = technical.length ? `<details><summary>technical error</summary>${technical.join('')}</details>` : '';
  return `<!--err:${index}-->${main}${details}<!--/err:${index}-->`;
}

// Replaces the cell of row `index` in an existing summary.html. Returns null when the run has no markers (older run).
export function replaceErrorCell(html: string, row: ErrorCellRow, index: number): string | null {
  const re = new RegExp(`<!--err:${index}-->[\\s\\S]*?<!--/err:${index}-->`);
  return re.test(html) ? html.replace(re, () => renderErrorCell(row, index)) : null;
}
```
`tests/support/results-reporter.ts` (plain reporter, no AI; writes `reports/<run>/results.csv`, `results.json`, `summary.html`, and `reports/index.html`):
```ts
import fs from 'node:fs';
import path from 'node:path';
import type { FullConfig, FullResult, Reporter, Suite, TestCase } from '@playwright/test/reporter';
import { csvPath, readCsvRows, slugify, splitRoles } from './roles';
import { describeErrors } from './error-text';
import { renderErrorCell } from './error-cell';

// Plain reporter, no AI. Writes one row per CSV test case and role:
// <outputDir>/results.csv, results.json, summary.html
type Status =
  | 'passed' | 'flaky' | 'failed' | 'timed out' | 'interrupted' | 'fixme' | 'skipped'
  | 'blocked' | 'not run' | 'not automated' | 'needs clarification';

type ResultRow = {
  testCaseId: string;
  scenario: string;
  expectedResult: string;
  role: string;
  status: Status;
  durationMs: number;
  retries: number;
  failedAt: string;
  error: string;
  technicalError: string;
  aiDescription?: string;
  aiModel?: string;
  specFile: string;
  trace: string;
  screenshot: string;
  video: string;
  testId: string;
};

type Options = { outputDir?: string; htmlReportDir?: string };
type RunReport = { run: Record<string, unknown>; totals: Record<string, number> };

const EXECUTED: Status[] = ['passed', 'flaky', 'failed', 'timed out', 'interrupted'];
const PASSED: Status[] = ['passed', 'flaky'];
const STEP_COMMENT = /^\s*\/\/\s*((?:A\d+|\d+)\.\s.*)$/;
const ANSI = /\u001b\[[0-9;]*m/g;

export default class ResultsReporter implements Reporter {
  private rootDir = process.cwd();
  private outputDir: string;
  private htmlReportDir: string;
  private suite?: Suite;
  private config?: FullConfig;
  private startedAt = new Date();
  private sources = new Map<string, string[]>();

  constructor(options: Options = {}) {
    this.outputDir = options.outputDir ?? 'reports';
    this.htmlReportDir = options.htmlReportDir ?? 'playwright-report';
  }

  printsToStdio() {
    return false;
  }

  onBegin(config: FullConfig, suite: Suite) {
    this.config = config;
    // Project root = folder of playwright.config.ts (config.rootDir is the test folder).
    this.rootDir = config.configFile ? path.dirname(config.configFile) : process.cwd();
    this.suite = suite;
    this.startedAt = new Date();
  }

  onEnd(result: FullResult) {
    // Nothing executed (listing, or every test filtered out): no run folder.
    if (!this.suite?.allTests().some((t) => t.results.length)) return;
    const csvFile = path.resolve(this.rootDir, csvPath());
    const csvRows = this.readCsv(csvFile);
    const authFailures: { role: string; slug: string; error: string }[] = [];
    const rows: ResultRow[] = [];

    for (const test of this.suite?.allTests() ?? []) {
      const project = test.parent.project();
      const projectName = project?.name ?? '';
      if (projectName.startsWith('setup-')) {
        if (test.outcome() === 'unexpected') {
          const slug = projectName.slice('setup-'.length);
          const name = (project?.metadata?.role as { name?: string } | undefined)?.name ?? slug;
          authFailures.push({ role: name, slug, error: this.errorOf(test).error });
        }
        continue;
      }
      if (test.tags.includes('@seed')) continue;

      const [testCaseId, ...rest] = test.title.trim().split(/\s+/);
      const csv = csvRows.get(testCaseId);
      const last = test.results.at(-1);
      const { error, technicalError, failedAt } = this.errorOf(test);
      rows.push({
        testCaseId,
        scenario: csv?.scenario ?? rest.join(' '),
        expectedResult: csv?.expected ?? '',
        role: csv?.roles.find((r) => r.slug === projectName)?.name ?? projectName,
        status: this.statusOf(test),
        durationMs: test.results.reduce((sum, r) => sum + r.duration, 0),
        retries: Math.max(test.results.length - 1, 0),
        failedAt,
        error,
        technicalError,
        specFile: path.relative(this.rootDir, test.location.file),
        trace: this.attachment(last?.attachments, 'trace'),
        screenshot: this.attachment(last?.attachments, 'screenshot'),
        video: this.attachment(last?.attachments, 'video'),
        testId: test.id,
      });
    }

    // Tests skipped because their role's sign-in setup failed.
    const failedRoles = new Set(authFailures.map((f) => f.slug));
    for (const row of rows) {
      const slug = slugify(row.role);
      if ((row.status === 'skipped' || row.status === 'not run') && failedRoles.has(slug)) {
        row.status = 'blocked';
        row.error = `Sign-in setup failed for role "${row.role}"`;
      }
    }

    // CSV rows with no test in this run.
    const clarification = this.read(path.resolve(this.rootDir, 'test-data/needs-clarification.md'));
    const csvStem = path.basename(csvFile, path.extname(csvFile));
    for (const [id, csv] of csvRows) {
      for (const role of csv.roles) {
        if (rows.some((r) => r.testCaseId === id && slugify(r.role) === role.slug)) continue;
        const spec = path.join('tests', csvStem, `${slugify(id)}.spec.ts`);
        const status: Status = clarification.includes(`| ${id} |`)
          ? 'needs clarification'
          : fs.existsSync(path.resolve(this.rootDir, spec)) ? 'not run' : 'not automated';
        rows.push({
          testCaseId: id, scenario: csv.scenario, expectedResult: csv.expected, role: role.name, status,
          durationMs: 0, retries: 0, failedAt: '', error: '', technicalError: '', specFile: status === 'not run' ? spec : '',
          trace: '', screenshot: '', video: '', testId: '',
        });
      }
    }

    rows.sort((a, b) =>
      a.testCaseId.localeCompare(b.testCaseId, undefined, { numeric: true }) || a.role.localeCompare(b.role));

    const totals: Record<string, number> = { total: rows.length };
    for (const row of rows) totals[row.status] = (totals[row.status] ?? 0) + 1;
    const executed = rows.filter((r) => EXECUTED.includes(r.status)).length;
    const passed = rows.filter((r) => PASSED.includes(r.status)).length;

    const report = {
      schemaVersion: 1,
      run: {
        startedAt: this.startedAt.toISOString(),
        finishedAt: new Date().toISOString(),
        durationMs: Math.round(result.duration),
        status: result.status,
        playwrightVersion: this.config?.version ?? '',
        workers: this.config?.workers ?? 0,
        baseUrlOrigin: process.env.BASE_URL ? new URL(process.env.BASE_URL).origin : '',
        csv: path.relative(this.rootDir, csvFile),
        passRate: executed ? Math.round((passed / executed) * 1000) / 10 : null,
      },
      totals,
      authFailures,
      results: rows,
    };

    const out = path.resolve(this.rootDir, this.outputDir);
    fs.mkdirSync(out, { recursive: true });
    fs.writeFileSync(path.join(out, 'results.json'), JSON.stringify(report, null, 2));
    fs.writeFileSync(path.join(out, 'results.csv'), toCsv(rows));
    fs.writeFileSync(path.join(out, 'summary.html'), this.toHtml(report, out));
    this.writeIndex(path.dirname(out));
  }

  // reports/index.html: every run folder, newest first. REPORTS_KEEP=<n> deletes the oldest beyond n.
  private writeIndex(root: string) {
    const isRun = (name: string) => /^\d{4}-\d{2}-\d{2}_\d{2}-\d{2}-\d{2}(-\d+)?$/.test(name);
    let runs = fs.readdirSync(root).filter((name) => isRun(name) && fs.existsSync(path.join(root, name, 'results.json'))).sort().reverse();
    const keep = Number(process.env.REPORTS_KEEP);
    if (Number.isInteger(keep) && keep > 0) {
      for (const old of runs.slice(keep)) fs.rmSync(path.join(root, old), { recursive: true, force: true });
      runs = runs.slice(0, keep);
    }
    const rows = runs.map((name) => {
      let report: RunReport | undefined;
      try {
        report = JSON.parse(fs.readFileSync(path.join(root, name, 'results.json'), 'utf8')) as RunReport;
      } catch {
        // Unreadable results.json: list the run without totals.
      }
      const t = report?.totals ?? {};
      const bad = (t['failed'] ?? 0) + (t['timed out'] ?? 0) + (t['interrupted'] ?? 0) + (t['blocked'] ?? 0);
      const passRate = report?.run.passRate;
      return `<tr><td><a href="${esc(name)}/summary.html"><b>${esc(name)}</b></a></td>
<td><span class="badge ${report ? (report.run.status === 'passed' ? 'ok' : 'bad') : 'na'}">${esc(String(report?.run.status ?? 'no results'))}</span></td>
<td class="num">${passRate === null || passRate === undefined ? '' : esc(String(passRate)) + '%'}</td>
<td class="num">${t['total'] ?? ''}</td><td class="num">${(t['passed'] ?? 0) + (t['flaky'] ?? 0) || ''}</td>
<td class="num">${bad || ''}</td><td class="num">${t['fixme'] ?? ''}</td>
<td class="num">${report ? ((Number(report.run.durationMs) || 0) / 1000).toFixed(1) + ' s' : ''}</td>
<td><a href="${esc(name)}/results.csv">csv</a> <a href="${esc(name)}/results.json">json</a> <a href="${esc(name)}/playwright-report/index.html">playwright</a></td></tr>`;
    }).join('\n');
    fs.writeFileSync(path.join(root, 'index.html'), `<!doctype html>
<html lang="en"><head><meta charset="utf-8"><title>Test runs</title>
<style>
body{font:14px/1.45 system-ui,-apple-system,Segoe UI,sans-serif;margin:24px;color:#1f2328}h1{font-size:20px}
table{border-collapse:collapse}th,td{border-bottom:1px solid #d1d9e0;padding:6px 12px;text-align:left}th{background:#f6f8fa}
.num{text-align:right}.badge{border-radius:10px;padding:1px 8px;font-size:12px}
.ok{background:#dafbe1;color:#116329}.bad{background:#ffebe9;color:#a40e26}.na{background:#eaeef2;color:#59636e}
</style></head><body><h1>Test runs (${runs.length})</h1>
<table><thead><tr><th>Run</th><th>Status</th><th>Pass rate</th><th>Rows</th><th>Passed</th><th>Failed / blocked</th><th>Fixme</th><th>Time</th><th>Files</th></tr></thead>
<tbody>
${rows}
</tbody></table></body></html>
`);
  }

  private statusOf(test: TestCase): Status {
    const last = test.results.at(-1);
    const annotations = [...test.annotations, ...(last?.annotations ?? [])];
    if (annotations.some((a) => a.type === 'fixme')) return 'fixme';
    if (!last) return 'not run';
    const outcome = test.outcome();
    if (outcome === 'expected') return 'passed';
    if (outcome === 'flaky') return 'flaky';
    if (outcome === 'skipped') return 'skipped';
    if (last.status === 'timedOut') return 'timed out';
    if (last.status === 'interrupted') return 'interrupted';
    return 'failed';
  }

  private errorOf(test: TestCase) {
    const last = test.results.at(-1);
    const first = last?.errors[0];
    if (!first || test.outcome() === 'expected' || test.outcome() === 'flaky') return { error: '', technicalError: '', failedAt: '' };
    const message = redact((first.message ?? first.value ?? '').replace(ANSI, ''));
    // Original Playwright text, kept for the "technical error" column.
    // A timeout names the missing element only in a later error, so every error contributes its first lines.
    const technicalError = (last?.errors ?? [first])
      .map((e) => redact((e.message ?? e.value ?? '').replace(ANSI, '')).split('\n').filter((l) => l.trim()).slice(0, 3).join(' | '))
      .filter(Boolean)
      .join(' || ')
      .slice(0, 700);
    // Plain sentence from every error of the last attempt (a timeout names the missing element in a later error).
    const readable = describeErrors((last?.errors ?? []).map((e) => redact(e.message ?? e.value ?? '')));
    // Timeouts put the step location on a later error.
    const location = last?.errors.find((e) => e.location)?.location;
    return { error: (readable || technicalError).slice(0, 500), technicalError, failedAt: location ? this.stepAt(location.file, location.line) : '' };
  }

  // Nearest "// A<n>. ..." or "// <n>. ..." comment above the failing line (see test-conventions.md).
  private stepAt(file: string, line: number) {
    if (!this.sources.has(file)) this.sources.set(file, this.read(file).split('\n'));
    const lines = this.sources.get(file)!;
    for (let i = Math.min(line, lines.length) - 1; i >= 0; i--) {
      const match = lines[i].match(STEP_COMMENT);
      if (match) return match[1].trim();
      if (/^\s*test(\.describe)?\(/.test(lines[i])) break;
    }
    return `line ${line}`;
  }

  private attachment(attachments: { name: string; path?: string }[] | undefined, name: string) {
    const found = attachments?.find((a) => a.name === name && a.path);
    return found?.path ? path.relative(this.rootDir, found.path) : '';
  }

  private readCsv(file: string) {
    const map = new Map<string, { scenario: string; expected: string; roles: ReturnType<typeof splitRoles> }>();
    try {
      for (const row of readCsvRows(file)) {
        const id = row['test case id'];
        if (id && !map.has(id)) {
          map.set(id, { scenario: row['test scenario'] ?? '', expected: row['expected result'] ?? '', roles: splitRoles(row['roles']) });
        }
      }
    } catch {
      // CSV missing or unreadable: report on executed tests only.
    }
    return map;
  }

  private read(file: string) {
    try {
      return fs.readFileSync(file, 'utf8');
    } catch {
      return '';
    }
  }

  private toHtml(report: { run: Record<string, unknown>; totals: Record<string, number>; authFailures: { role: string; slug: string; error: string }[]; results: ResultRow[] }, out: string) {
    const link = (target: string) => esc(path.relative(out, path.resolve(this.rootDir, target)).split(path.sep).join('/'));
    const htmlReport = link(path.join(this.htmlReportDir, 'index.html'));
    const { run, totals } = report;
    const cards = ['total', 'passed', 'flaky', 'failed', 'timed out', 'fixme', 'blocked', 'skipped', 'not run', 'not automated', 'needs clarification']
      .filter((k) => k === 'total' || totals[k])
      .map((k) => `<div class="card ${cls(k)}"><b>${totals[k] ?? 0}</b><span>${esc(k)}</span></div>`)
      .join('');
    const auth = report.authFailures.length
      ? `<div class="alert"><b>Sign-in setup failed:</b><ul>${report.authFailures
        .map((f) => `<li>${esc(f.role)}: ${esc(f.error)}</li>`).join('')}</ul></div>`
      : '';
    const body = report.results.map((r, i) => `<tr>
<td><b>${esc(r.testCaseId)}</b></td><td>${esc(r.scenario)}</td><td>${esc(r.expectedResult)}</td><td>${esc(r.role)}</td>
<td><span class="badge ${cls(r.status)}">${esc(r.status)}</span>${r.retries ? ` <small>(${r.retries} retr${r.retries > 1 ? 'ies' : 'y'})</small>` : ''}</td>
<td class="num">${r.durationMs ? (r.durationMs / 1000).toFixed(1) + ' s' : ''}</td>
<td>${r.failedAt ? `<b>${esc(r.failedAt)}</b><br>` : ''}${renderErrorCell(r, i)}</td>
<td class="links">${r.testId ? `<a href="${htmlReport}#?testId=${esc(r.testId)}">details</a>` : ''}${r.screenshot ? ` <a href="${link(r.screenshot)}">screenshot</a>` : ''}${r.video ? ` <a href="${link(r.video)}">video</a>` : ''}</td>
</tr>`).join('\n');
    return `<!doctype html>
<html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Test results ${esc(String(run.startedAt))}</title>
<style>
body{font:14px/1.45 system-ui,-apple-system,Segoe UI,sans-serif;margin:24px;color:#1f2328}
h1{font-size:20px;margin:0 0 4px}.meta{color:#59636e;margin-bottom:16px}
.cards{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:16px}
.card{border:1px solid #d1d9e0;border-radius:8px;padding:8px 14px;min-width:84px}
.card b{display:block;font-size:22px}.card span{color:#59636e}
.alert{background:#fff1f0;border:1px solid #ffa39e;border-radius:8px;padding:8px 14px;margin-bottom:16px}
table{border-collapse:collapse;width:100%}th,td{border-bottom:1px solid #d1d9e0;padding:6px 8px;text-align:left;vertical-align:top}
th{background:#f6f8fa;position:sticky;top:0}td:first-child,.num,.links{white-space:nowrap}.num{text-align:right}
code{font-size:12px;color:#82071e;word-break:break-word}
.badge{border-radius:10px;padding:1px 8px;font-size:12px;white-space:nowrap;background:#eaeef2}
.ok{background:#dafbe1;color:#116329}.warn{background:#fff8c5;color:#7d4e00}.bad{background:#ffebe9;color:#a40e26}.na{background:#eaeef2;color:#59636e}
.card.ok,.card.warn,.card.bad,.card.na{background:none}.card.ok b{color:#116329}.card.warn b{color:#9a6700}.card.bad b{color:#cf222e}
</style></head><body>
<h1>Test results</h1>
<div class="meta">${esc(String(run.startedAt))} · ${((Number(run.durationMs) || 0) / 1000).toFixed(1)} s · run ${esc(String(run.status))}
· pass rate ${run.passRate === null ? 'n/a' : esc(String(run.passRate)) + '%'} of executed · ${esc(String(run.workers))} workers
· Playwright ${esc(String(run.playwrightVersion))} · CSV ${esc(String(run.csv))}${run.baseUrlOrigin ? ' · ' + esc(String(run.baseUrlOrigin)) : ''}
· <a href="${htmlReport}">full Playwright report</a></div>
<div class="cards">${cards}</div>
${auth}
<table><thead><tr><th>TC-ID</th><th>Scenario</th><th>Expected result</th><th>Role</th><th>Status</th><th>Time</th><th>Failed at / error</th><th></th></tr></thead>
<tbody>
${body}
</tbody></table>
</body></html>
`;
  }
}

function cls(status: string) {
  if (status === 'passed') return 'ok';
  if (status === 'flaky' || status === 'fixme' || status === 'needs clarification') return 'warn';
  if (status === 'failed' || status === 'timed out' || status === 'interrupted' || status === 'blocked') return 'bad';
  return status === 'total' ? '' : 'na';
}

function esc(value: string) {
  return value.replace(/[&<>"']/g, (c) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' })[c]!);
}

// Never let a role password from .env reach a report.
function redact(text: string) {
  let out = text;
  for (const [key, value] of Object.entries(process.env)) {
    if (/^ROLE_.+_PASSWORD$/.test(key) && value) out = out.split(value).join('***');
  }
  return out;
}

function toCsv(rows: ResultRow[]) {
  const header = ['test case id', 'test scenario', 'expected result', 'role', 'status', 'duration (s)', 'retries',
    'failed at', 'error', 'spec file', 'trace', 'screenshot', 'video', 'technical error', 'description by'];
  const quote = (v: string | number) => `"${String(v).replace(/"/g, '""')}"`;
  const lines = rows.map((r) => [r.testCaseId, r.scenario, r.expectedResult, r.role, r.status,
  r.durationMs ? (r.durationMs / 1000).toFixed(1) : '', r.retries, r.failedAt, r.aiDescription || r.error, r.specFile, r.trace, r.screenshot, r.video, r.technicalError,
    r.error || r.aiDescription ? (r.aiDescription ? (r.aiModel ?? 'AI') : 'rules') : '']
    .map(quote).join(','));
  // BOM so Excel opens UTF-8 correctly.
  return '\uFEFF' + [header.map(quote).join(','), ...lines].join('\r\n') + '\r\n';
}
```
`tests/support/describe-failures.ts` (optional post-run step, never run by `playwright test`; see `describe.md`. Needs `@aws/bedrock-token-generator`):
```ts
// Post-run step (never part of a test run): asks Gemma 4 E2B on Amazon Bedrock to describe each failed test in plain
// words and adds the text to a run folder as an "AI description".
//
//   bun run describe                                 latest run in reports/
//   bun run describe -- reports/<run>                a given run
//   bun run describe -- --profile <name>             AWS profile to use (or set DESCRIBE_AWS_PROFILE in .env)
//   bun run describe -- --region us-west-2           Bedrock region (or set DESCRIBE_REGION in .env), default us-east-1
//   bun run describe -- --force                      describe rows that already have an AI description
//   bun run describe -- --dry-run                    print what would be sent, call nothing (no profile needed)
//
// AWS profile (required): the profile is never guessed. Set DESCRIBE_AWS_PROFILE in .env or pass --profile <name>.
// Without one the script stops before sending anything; ask the agent which profile to use.
// Region: Gemma 4 E2B is served in us-east-1, us-east-2, us-west-2 and eu-central-1 (not ap-southeast-1). The short-term
// token is only valid in the region it was made for.
// Permissions of the profile: bedrock-mantle:CreateInference and bedrock-mantle:CallWithBearerToken
// (managed policy AmazonBedrockMantleInferenceAccess).
//
// Files changed in the run folder: results.json (aiDescription, aiModel), results.csv (the "error" column becomes the AI
// description and "description by" names the model), summary.html (the AI description is the visible text; the rule-based
// sentence and the original Playwright error move into the collapsed "technical error"). Pass/fail is never touched.
//
// Sent to the model: test case id, scenario, expected result, role, failing step, the report's error text and the
// clarification notes of the spec. Not sent: screenshots, traces, page content, .env or credentials.
// Auth: the credentials of the chosen profile, turned into a short-term (15 min) Bedrock bearer token for one region.
// Docs: https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-google-gemma-4-e2b.html
import fs from 'node:fs';
import path from 'node:path';
import { parse } from 'csv-parse/sync';
import { getTokenProvider } from '@aws/bedrock-token-generator';
import { replaceErrorCell } from './error-cell';

// Same .env as playwright.config.ts. Bun already loads .env on its own and has no process.loadEnvFile; Node needs the call.
if (fs.existsSync('.env') && typeof process.loadEnvFile === 'function') process.loadEnvFile('.env');

const MODEL = 'google.gemma-4-e2b';
const FAILED = new Set(['failed', 'timed out', 'interrupted']);

type Row = {
  testCaseId: string;
  scenario: string;
  expectedResult: string;
  role: string;
  status: string;
  failedAt: string;
  error: string;
  technicalError?: string;
  aiDescription?: string;
  aiModel?: string;
};

const args = process.argv.slice(2);
const flag = (name: string) => args.includes(name);
const option = (name: string) => {
  const i = args.indexOf(name);
  const value = i >= 0 ? args[i + 1] : undefined;
  return value && !value.startsWith('--') ? value : undefined;
};
const OPTIONS_WITH_VALUE = ['--region', '--profile'];
const REGION = option('--region') || process.env.DESCRIBE_REGION || 'us-east-1'; // us-east-1 is a supported region
const PROFILE = (option('--profile') || process.env.DESCRIBE_AWS_PROFILE || '').trim();
const positional = args.filter((a, i) => !a.startsWith('--') && !OPTIONS_WITH_VALUE.includes(args[i - 1]));

function latestRun(): string {
  const dirs = fs.existsSync('reports')
    ? fs.readdirSync('reports').filter((n) => /^\d{4}-\d{2}-\d{2}_\d{2}-\d{2}-\d{2}(-\d+)?$/.test(n) && fs.existsSync(path.join('reports', n, 'results.json'))).sort()
    : [];
  if (!dirs.length) throw new Error('No run folder with results.json under reports/. Run the tests first.');
  return path.join('reports', dirs[dirs.length - 1]);
}

// Spec notes that explain what the test case expects (owner decisions), without steps or data.
function specNotes(testCaseId: string): string {
  const file = path.join('specs', `${testCaseId}.md`);
  if (!fs.existsSync(file)) return '';
  return fs
    .readFileSync(file, 'utf8')
    .split('\n')
    .filter((l) => /^- Clarification applied:/.test(l))
    .map((l) => l.replace(/^- Clarification applied:\s*/, '- '))
    .join('\n')
    .slice(0, 1500);
}

const SYSTEM = `You write short notes for a software tester who reads an automated test report.
You get one failed test: what it should check, the step where it stopped, and the error text.
Write 2 or 3 plain sentences, no markdown, no bullet points, no code:
1. What the test was doing and what it expected.
2. What happened instead, using only facts in the error text.
Rules: do not guess the cause, do not say whether it is a product defect or a test problem, do not invent screen details.
If the notes mention an owner decision (like C-3), you may name it once. Keep field and button names exactly as given.
In "expected X, got Y" the test expected X and the page showed Y.
Example 1: "The test tried to type a new value into 'Redemption Amount (MYR)' on the Documentation edit page, as the test case expects the field to be editable. It could not find a text box for that field, so the test ran out of time and the later checks did not run."
Example 2: "The test entered a new 'New Redemption Amount (MYR)' of 90000.00 and expected the page to show it. The page still showed 80000, both on the edit page and on the summary after saving."`;

function userPrompt(r: Row): string {
  return [
    `Test case: ${r.testCaseId} ${r.scenario}`,
    `Role: ${r.role}`,
    `Expected result (from the test case): ${r.expectedResult}`,
    `Step where it stopped: ${r.failedAt || 'unknown'}`,
    `Result: ${r.status}`,
    `Error text from the report: ${r.error}`,
    specNotes(r.testCaseId) ? `Notes on this test case:\n${specNotes(r.testCaseId)}` : '',
  ].filter(Boolean).join('\n');
}

async function ask(token: string, prompt: string, maxTokens: number): Promise<{ text?: string; note?: string }> {
  const res = await fetch(`https://bedrock-mantle.${REGION}.api.aws/openai/v1/chat/completions`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
    body: JSON.stringify({
      model: MODEL,
      messages: [{ role: 'system', content: SYSTEM }, { role: 'user', content: prompt }],
      // Reasoning tokens count against max_tokens; "high" keeps the reasoning out of the answer text (model card).
      reasoning_effort: 'high',
      max_tokens: maxTokens,
      temperature: 0.2,
    }),
    signal: AbortSignal.timeout(90_000),
  });
  if (!res.ok) return { note: `HTTP ${res.status}: ${(await res.text()).slice(0, 200)}` };
  const body = (await res.json()) as { choices?: { finish_reason?: string; message?: { content?: string | null } }[] };
  const choice = body.choices?.[0];
  const text = choice?.message?.content?.trim();
  if (text) return { text };
  return { note: choice?.finish_reason === 'length' ? 'length' : 'empty reply' };
}

async function describe(token: string, row: Row): Promise<string | undefined> {
  const prompt = userPrompt(row);
  for (const maxTokens of [3000, 8000]) {
    for (let attempt = 0; attempt < 2; attempt++) {
      try {
        const { text, note } = await ask(token, prompt, maxTokens);
        if (text) return text.replace(/\s+/g, ' ').replace(/[*_`#]/g, '');
        if (note === 'length') break; // reasoning used the whole budget: retry once with a larger one
        if (attempt === 0 && /HTTP (429|5\d\d)/.test(note ?? '')) continue;
        console.warn(`  ${row.testCaseId} ${row.role}: ${note}`);
        return undefined;
      } catch (e) {
        if (attempt === 1) {
          console.warn(`  ${row.testCaseId} ${row.role}: ${e instanceof Error ? e.message : e}`);
          return undefined;
        }
      }
    }
  }
  console.warn(`  ${row.testCaseId} ${row.role}: no answer within the token budget`);
  return undefined;
}

// results.csv rows are written in the same order as results.json, so row i of one is row i of the other.
// The "error" column holds the AI description when there is one (the rule-based sentence stays in results.json and
// in the collapsed technical details of summary.html); "description by" says which one it is.
function patchCsv(file: string, rows: Row[]) {
  const records = parse(fs.readFileSync(file), { bom: true, relax_column_count: true }) as string[][];
  const header = records[0];
  const col = (name: string) => {
    let i = header.indexOf(name);
    if (i < 0) {
      header.push(name);
      i = header.length - 1;
    }
    return i;
  };
  const errorCol = col('error');
  const byCol = col('description by');
  records.slice(1).forEach((record, i) => {
    const row = rows[i];
    if (!row) return;
    while (record.length < header.length) record.push('');
    if (row.aiDescription) {
      record[errorCol] = row.aiDescription;
      record[byCol] = row.aiModel ?? MODEL;
    }
  });
  const quote = (v: string) => `"${String(v).replace(/"/g, '""')}"`;
  // BOM so Excel opens UTF-8 correctly (same as the reporter).
  fs.writeFileSync(file, '\uFEFF' + records.map((r) => r.map(quote).join(',')).join('\r\n') + '\r\n');
}

// Swaps the error cell of every described row. Run folders from before the cell markers existed are left alone.
function patchHtml(file: string, rows: Row[]) {
  let html = fs.readFileSync(file, 'utf8');
  let skipped = 0;
  rows.forEach((row, i) => {
    if (!row.aiDescription) return;
    const next = replaceErrorCell(html, row, i);
    if (next === null) skipped++;
    else html = next;
  });
  if (skipped) console.warn(`  summary.html has no error-cell markers for ${skipped} row(s) (run made before this version); rerun the tests to get them.`);
  fs.writeFileSync(file, html);
}

async function main() {
  // No fallback to the default credential chain: the user must name the profile. A dry run sends nothing, so it is exempt.
  if (!PROFILE && !flag('--dry-run')) {
    throw new Error(
      'No AWS profile set. Nothing was sent. Set DESCRIBE_AWS_PROFILE in .env (or pass --profile <name>) to the AWS profile ' +
      'you want to use for the AI descriptions, and ask the agent which profile that should be.',
    );
  }
  const run = positional[0] ?? latestRun();
  const jsonFile = path.join(run, 'results.json');
  const report = JSON.parse(fs.readFileSync(jsonFile, 'utf8')) as { results: Row[] };
  const failed = report.results.filter((r) => FAILED.has(r.status));
  console.log(`Run ${run}: ${failed.length} failed row(s). Model ${MODEL}.`);
  console.log(`AWS profile: ${PROFILE || '(not set)'}, region: ${REGION}`);
  if (!failed.length) return;

  if (flag('--dry-run')) {
    for (const r of failed) console.log(`\n--- ${r.testCaseId} ${r.role}\n${userPrompt(r)}`);
    return;
  }

  console.log(`Sending the text of the failed rows (no screenshots) to Amazon Bedrock in ${REGION} with AWS profile ${PROFILE}.`);
  let token: string;
  try {
    token = await getTokenProvider({ profile: PROFILE, region: REGION, expiresInSeconds: 900 })();
  } catch (e) {
    throw new Error(`Could not get AWS credentials for profile "${PROFILE}": ${e instanceof Error ? e.message : e}`);
  }

  let done = 0;
  for (const row of failed) {
    if (row.aiDescription && !flag('--force')) continue;
    const text = await describe(token, row);
    if (text) {
      row.aiDescription = text;
      row.aiModel = MODEL;
      done++;
      console.log(`  ${row.testCaseId} ${row.role}: ${text}`);
    }
  }

  fs.writeFileSync(jsonFile, JSON.stringify(report, null, 2));
  patchCsv(path.join(run, 'results.csv'), report.results);
  patchHtml(path.join(run, 'summary.html'), report.results);
  console.log(`Done: ${done} new description(s) written to ${run}.`);
}

main().catch((e) => {
  console.error(e instanceof Error ? e.message : e);
  process.exit(1);
});
```
`tests/auth.setup.ts`:

```ts
import fs from 'node:fs';
import { test as setup, expect, type Browser, type Page } from '@playwright/test';
import { type Role, authFile, credentialsFor, sessionFile } from './support/roles';
import { appUrlPattern, expectSignedIn } from './support/app';

const MAX_AGE_MINUTES = Number(process.env.AUTH_MAX_AGE_MINUTES ?? 45);
const LOGIN_TIMEOUT = 60_000;

// One setup project per role runs this file; the role comes from project metadata.
setup('authenticate', async ({ page, browser }) => {
  const role = setup.info().project.metadata.role as Role;
  const file = authFile(role.slug);
  if (isFresh(file) && (await stillSignedIn(browser, file))) return;

  await signInWithMicrosoft(page, role);
  await expectSignedIn(page);
  fs.mkdirSync('.auth', { recursive: true });
  await page.context().storageState({ path: file, indexedDB: true });
  fs.writeFileSync(sessionFile(role.slug), await page.evaluate(() => JSON.stringify(sessionStorage)));
});

function isFresh(file: string) {
  return fs.existsSync(file) && (Date.now() - fs.statSync(file).mtimeMs) / 60_000 < MAX_AGE_MINUTES;
}

async function stillSignedIn(browser: Browser, file: string) {
  const context = await browser.newContext({ storageState: file, baseURL: process.env.BASE_URL, locale: 'en-US' });
  try {
    const page = await context.newPage();
    await page.goto('/');
    await expectSignedIn(page);
    return true;
  } catch {
    return false;
  } finally {
    await context.close();
  }
}

// Microsoft Entra ID sign-in, no 2FA.
// Email step verified live (Playwright 1.64.1, 2026-11-01).
// Password, "Use your password" and "Stay signed in?" steps: VERIFY via MCP on first run with a real test account.
async function signInWithMicrosoft(page: Page, role: Role) {
  const { username, password } = credentialsFor(role);
  await page.goto('/');

  await page.getByRole('textbox', { name: /email/i }).fill(username);
  await page.getByRole('button', { name: 'Next' }).click();

  const passwordBox = page.getByLabel(/password/i);
  const usePassword = page.getByRole('button', { name: 'Use your password' });
  await expect(passwordBox.or(usePassword).first()).toBeVisible({ timeout: LOGIN_TIMEOUT });
  if (await usePassword.isVisible()) await usePassword.click();
  await passwordBox.fill(password);
  await page.getByRole('button', { name: 'Sign in' }).click();

  // "Stay signed in?" may or may not appear. Answer Yes so Microsoft keeps a persistent session cookie.
  const staySignedIn = page.getByText('Stay signed in?');
  await Promise.race([
    page.waitForURL(appUrlPattern(), { timeout: LOGIN_TIMEOUT }),
    staySignedIn.waitFor({ timeout: LOGIN_TIMEOUT }),
  ]);
  if (await staySignedIn.isVisible()) await page.getByRole('button', { name: 'Yes' }).click();
  await page.waitForURL(appUrlPattern(), { timeout: LOGIN_TIMEOUT });
}
```

`tests/seed.spec.ts` (starts signed in through the role project's storageState):

```ts
import { test } from './support/fixtures';
import { expectSignedIn } from './support/app';

// Seed for the planner and generator. Run it under a role project
// (planner_setup_page / generator_setup_page with project: "<role-slug>").
// The role project loads .auth/<role-slug>.json, so the page starts signed in.
test.describe('Test group', () => {
  test('seed', { tag: '@seed' }, async ({ page }) => {
    await page.goto('/');
    await expectSignedIn(page);
    // generate code here.
  });
});
```

`.gitignore` (append missing lines only):

```gitignore
node_modules/
.env
.auth/
test-results/
playwright-report/
blob-report/
reports/
```

`.env.example`: write one `ROLE_<SLUG>_USERNAME` / `ROLE_<SLUG>_PASSWORD` pair per unique role in the CSV. `<SLUG>` is the role slug uppercased with `-` replaced by `_` (role `Credit Officer` becomes `CREDIT_OFFICER`). Leave every value empty.

```dotenv
BASE_URL=https://your-app.example.com
TEST_CASES_CSV=test-data/test-cases.csv
AUTH_MAX_AGE_MINUTES=45
# PW_WORKERS=4
# REPORTS_KEEP=30   # keep only the newest 30 run folders (default: keep all)
# One pair per role from the CSV "roles" column:
ROLE_<SLUG>_USERNAME=
ROLE_<SLUG>_PASSWORD=
# Optional AI descriptions of failed tests (describe.md). Names only, never keys. Leave empty until the user chooses.
DESCRIBE_AWS_PROFILE=
DESCRIBE_REGION=
```

`package.json` scripts (add if missing):

```json
{
  "scripts": {
    "test": "playwright test --grep-invert @seed",
    "test:role": "playwright test --grep-invert @seed --project",
    "report": "playwright show-report \"reports/$(ls reports | grep '^20' | tail -1)/playwright-report\"",
    "describe": "bun tests/support/describe-failures.ts"
  }
}
```

The `describe` script is optional and needs bun at run time. Tests do not.
Create the empty folder `specs/`.

## Step 5: Verify the scaffold

1. `bunx playwright test --list` must show one `setup-<role>` and one `<role>` project per CSV role.
2. Ask the user to copy `.env.example` to `.env` and fill it. Wait for confirmation.
3. Follow `microsoft-auth.md` to verify the sign-in locators and add the signed-in landmark.
4. `bunx playwright test tests/seed.spec.ts` must pass for every role.
5. Optional, after step 4 passes: ask the user about AI descriptions of failed tests and follow `describe.md`. Skip it if the user declines. Do not set `DESCRIBE_AWS_PROFILE` yourself.

More run commands: `--only-changed=main` (specs changed vs main), `--shard=1/4` (CI).

Speed defaults: `fullyParallel`, headless Chromium, sessions reused for `AUTH_MAX_AGE_MINUTES`, trace, screenshot and video recorded for every test (setup projects off), no local retries.
