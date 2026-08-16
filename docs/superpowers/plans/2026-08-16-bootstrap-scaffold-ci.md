# Bootstrap 1: Scaffold + CI + Ruleset Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A booting Next.js app in this repo with typecheck/lint/test/build enforced as required CI checks behind a ruleset with auto-merge on `main`.

**Architecture:** Next.js App Router scaffold (created in a side directory, merged in — the repo is non-empty), vitest + Testing Library, one CI workflow with four parallel jobs whose names are the ruleset's required check contexts. Server-side enforcement (ruleset + repo merge settings) lands last, after CI is green on main, so the repo is never wedged behind checks that don't exist yet.

**Tech Stack:** pnpm, Next.js (current stable at execution — accept scaffold defaults), TypeScript strict, Tailwind, ESLint, Prettier, vitest + @testing-library/react, GitHub Actions, GitHub rulesets via `gh api`.

## Global Constraints

- ADR 0001 is the SSOT; do not deviate without a superseding ADR.
- Repo is **public**: never write a secret into any file; `.env*` stays gitignored.
- All files LF (`.gitattributes` already enforces).
- Work happens in `D:\claude\projects\ma-ops` (flat, no nesting).
- Until the ruleset lands (Task 6), commits go straight to `main` — this is the only plan allowed to do that; from Task 6 onward, PRs only.
- Node + pnpm on PATH: on the operator box prefix `C:\Program Files\nodejs` for non-interactive shells.

---

### Task 1: Next.js scaffold merged into the repo

**Files:**
- Create: `package.json`, `next.config.ts`, `tsconfig.json`, `src/app/*` (scaffold set), `postcss.config.mjs`, `eslint.config.mjs`
- Modify: `.gitignore` (merge scaffold entries if any are missing)

**Interfaces:**
- Produces: a `pnpm build`-able Next.js app; `package.json` scripts `dev`, `build`, `lint` that later tasks extend.

- [ ] **Step 1: Scaffold in a side directory** (create-next-app refuses non-empty dirs)

```bash
cd D:/claude/projects/ma-ops
pnpm dlx create-next-app@latest _scaffold --typescript --tailwind --eslint --app --src-dir --use-pnpm --no-import-alias --turbopack
```

Expected: `_scaffold/` created, dependencies installed, exit 0.

- [ ] **Step 2: Merge scaffold into repo root**

```bash
cd D:/claude/projects/ma-ops
mv _scaffold/.gitignore _scaffold/gitignore.scaffold
cp -r _scaffold/. .
rm -rf _scaffold
```

Then open `gitignore.scaffold`, append any lines missing from the existing `.gitignore` (do not remove existing entries — `CLAUDE.local.json`, `.claude/settings.local.json` lines must survive), delete `gitignore.scaffold`. The scaffold's `README.md` must NOT overwrite ours — if it did, `git checkout -- README.md`.

- [ ] **Step 3: Strict TypeScript**

In `tsconfig.json` confirm `"strict": true` (scaffold default). Add if absent:

```json
"noUncheckedIndexedAccess": true
```

- [ ] **Step 4: Verify build**

```bash
pnpm build
```

Expected: `✓ Compiled successfully`, exit 0.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: Next.js scaffold (TS strict, Tailwind, App Router)" && git push
```

### Task 2: Vitest + first test

**Files:**
- Create: `vitest.config.ts`, `src/app/page.test.tsx`
- Modify: `package.json` (scripts + devDeps)

**Interfaces:**
- Produces: `pnpm test` (vitest run) — the `test` CI job runs exactly this.

- [ ] **Step 1: Install**

```bash
pnpm add -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom vite-tsconfig-paths
```

- [ ] **Step 2: Config**

`vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    environment: "jsdom",
    include: ["src/**/*.test.{ts,tsx}"],
  },
});
```

`package.json` scripts — add:

```json
"test": "vitest run",
"typecheck": "tsc --noEmit"
```

- [ ] **Step 3: Write the failing test**

`src/app/page.test.tsx`:

```tsx
import { render, screen } from "@testing-library/react";
import { expect, test } from "vitest";
import Home from "./page";

test("home page renders the app name", () => {
  render(<Home />);
  expect(screen.getByText(/ma-ops/i)).toBeDefined();
});
```

- [ ] **Step 4: Run test to verify it fails**

```bash
pnpm test
```

Expected: FAIL — scaffold home page does not contain "ma-ops".

- [ ] **Step 5: Minimal implementation**

Replace the scaffold `src/app/page.tsx` content with:

```tsx
export default function Home() {
  return (
    <main className="flex min-h-screen items-center justify-center">
      <h1 className="text-2xl font-semibold">ma-ops</h1>
    </main>
  );
}
```

- [ ] **Step 6: Run test to verify it passes**

```bash
pnpm test
```

Expected: `1 passed`, exit 0.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "test: vitest + RTL harness with home smoke test" && git push
```

### Task 3: Prettier + format check

**Files:**
- Create: `.prettierrc.json`, `.prettierignore`
- Modify: `package.json` (scripts)

**Interfaces:**
- Produces: `pnpm format:check` — the `lint` CI job runs it beside `pnpm lint`. Repo-wide formatting is clean from this commit forever (ADR: drift can never accumulate).

- [ ] **Step 1: Install + config**

```bash
pnpm add -D prettier
```

`.prettierrc.json`:

```json
{}
```

(Defaults. Deviations need a reason recorded here when they happen.)

`.prettierignore`:

```
.next/
pnpm-lock.yaml
```

`package.json` scripts — add:

```json
"format": "prettier --write .",
"format:check": "prettier --check ."
```

- [ ] **Step 2: Format the whole repo once, verify check passes**

```bash
pnpm format && pnpm format:check
```

Expected: `All matched files use Prettier code style!`, exit 0.

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "chore: prettier + format-check, repo formatted" && git push
```

### Task 4: CI workflow — four required jobs

**Files:**
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Produces: job names `typecheck`, `lint`, `test`, `build` — Task 6's ruleset requires exactly these contexts. Renaming a job later means updating the ruleset in the same change.

- [ ] **Step 1: Write the workflow**

`.github/workflows/ci.yml`:

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]
jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm format:check
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
```

(Node 22 = current LTS at plan time — verify at execution and bump if LTS moved; `pnpm/action-setup@v4` reads the version from `packageManager` in `package.json`.)

- [ ] **Step 2: Push and verify all four jobs green**

```bash
git add -A && git commit -m "ci: typecheck, lint, test, build jobs" && git push
gh run watch --exit-status
```

Expected: run completes, all 4 jobs ✓, exit 0. If a job reds, fix before Task 5 — Task 6 will freeze these as required.

### Task 5: scripts/verify + pnpm doctor v0

**Files:**
- Create: `scripts/verify.mjs`, `scripts/doctor.mjs`
- Modify: `package.json` (scripts)

**Interfaces:**
- Produces: `pnpm verify` (the full local gate, structured `EXITCODE=` line, never piped) and `pnpm doctor` (prerequisite checker). Later plans extend both; the output contract (`EXITCODE=<n>` as the last line) is frozen here.

- [ ] **Step 1: Write scripts/verify.mjs**

```js
#!/usr/bin/env node
// Runs the full local gate. Prints a structured verdict; exit code is the truth.
// Never pipe this script (ADR 0001: pipes have eaten verdicts).
import { spawnSync } from "node:child_process";

const steps = [
  ["typecheck", ["pnpm", "typecheck"]],
  ["lint", ["pnpm", "lint"]],
  ["format", ["pnpm", "format:check"]],
  ["test", ["pnpm", "test"]],
  ["build", ["pnpm", "build"]],
];

let failed = [];
for (const [name, [cmd, ...args]] of steps) {
  console.log(`\n=== ${name} ===`);
  const r = spawnSync(cmd, args, { stdio: "inherit", shell: true });
  if (r.status !== 0) failed.push(name);
}
const code = failed.length === 0 ? 0 : 1;
console.log(`\nVERIFY ${code === 0 ? "GREEN" : `RED (${failed.join(", ")})`}`);
console.log(`EXITCODE=${code}`);
process.exit(code);
```

- [ ] **Step 2: Write scripts/doctor.mjs**

```js
#!/usr/bin/env node
// Prerequisite checker. ✅/❌ per item; exit 1 if anything ❌.
import { spawnSync } from "node:child_process";

const checks = [
  ["node >= 22", ["node", "--version"], (o) => Number(o.slice(1).split(".")[0]) >= 22],
  ["pnpm", ["pnpm", "--version"], (o) => o.trim().length > 0],
  ["git", ["git", "--version"], (o) => o.includes("git version")],
  ["gh authenticated", ["gh", "auth", "status"], (_, code) => code === 0],
  ["repo clean or known", ["git", "status", "--porcelain"], () => true],
];

let bad = 0;
for (const [label, [cmd, ...args], ok] of checks) {
  const r = spawnSync(cmd, args, { encoding: "utf8", shell: true });
  const pass = ok(r.stdout ?? "", r.status);
  console.log(`${pass ? "✅" : "❌"} ${label}`);
  if (!pass) bad++;
}
console.log(`EXITCODE=${bad === 0 ? 0 : 1}`);
process.exit(bad === 0 ? 0 : 1);
```

`package.json` scripts — add:

```json
"verify": "node scripts/verify.mjs",
"doctor": "node scripts/doctor.mjs"
```

- [ ] **Step 3: Run both, verify verdicts**

```bash
pnpm doctor
pnpm verify
```

Expected: doctor all ✅ `EXITCODE=0`; verify `VERIFY GREEN` `EXITCODE=0`.

Negative control (required — a verifier that cannot fail certifies nothing): break `src/app/page.tsx` (e.g. delete the export), run `pnpm verify`, expect `VERIFY RED` + `EXITCODE=1`; restore the file, re-run, expect GREEN.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "chore: verify + doctor scripts with structured verdicts" && git push
```

### Task 6: Ruleset + merge settings — enforcement on

**Files:** none (GitHub settings via `gh api`; record = this plan + ADR)

**Interfaces:**
- Consumes: CI job names from Task 4 (`typecheck`, `lint`, `test`, `build`).
- Produces: `main` accepts changes only via PR with those checks green and branch up-to-date; squash-only; auto-merge available. **From here on, this plan's direct-push allowance is dead.**

- [ ] **Step 1: Repo merge settings**

```bash
gh api -X PATCH repos/vap-ops/ma-ops -f allow_squash_merge=true -f allow_merge_commit=false -f allow_rebase_merge=false -f allow_auto_merge=true -f delete_branch_on_merge=true
```

Expected: JSON echo with `"allow_auto_merge": true`, exit 0.

- [ ] **Step 2: Create the ruleset**

```bash
gh api repos/vap-ops/ma-ops/rulesets --input - <<'JSON'
{
  "name": "main-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
  "bypass_actors": [],
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "required_linear_history" },
    { "type": "pull_request", "parameters": {
        "required_approving_review_count": 0,
        "dismiss_stale_reviews_on_push": false,
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": false,
        "allowed_merge_methods": ["squash"] } },
    { "type": "required_status_checks", "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          { "context": "typecheck" },
          { "context": "lint" },
          { "context": "test" },
          { "context": "build" } ] } }
  ]
}
JSON
```

Expected: HTTP 201 with the ruleset JSON. (`strict_required_status_checks_policy: true` = the require-up-to-date queue substitute from ADR §S3.)

- [ ] **Step 3: Prove the gate with a real PR (positive + negative control)**

```bash
git push origin main 2>&1 | head -3
```

Expected: **rejected** (ruleset). Then the first real PR:

```bash
git checkout -b chore/ruleset-canary
```

Edit `README.md` — change the Status section's first line to `Bootstrapped: scaffold + CI + ruleset live (plan 1).` Then:

```bash
git add -A && git commit -m "docs: ruleset canary — first PR through the gate"
git push -u origin chore/ruleset-canary
gh pr create --fill
gh pr merge --auto --squash
gh pr checks --watch
```

Expected: 4 checks pass → PR auto-merges → `gh pr view --json state` shows `MERGED`. Verify main moved: `git fetch && git log origin/main --oneline -1` shows the squash commit.

- [ ] **Step 4: Sync local**

```bash
git checkout main && git pull && git branch -D chore/ruleset-canary
```

---

## Self-review notes

- Spec coverage: ADR §Bootstrap step 1 fully; step 3 partially (ruleset + auto-merge here; AI review + danger-path = plan 3, `db` check = plan 2 — each adds its context to the ruleset's required list in its own plan).
- Direct-push-to-main window is explicit (Global Constraints + Task 6 kills it).
- Job names ↔ ruleset contexts cross-checked (typecheck/lint/test/build).
- Node 22 / action versions are execution-time-verifiable pins, flagged in Task 4.
