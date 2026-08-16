# ADR 0001 — Founding architecture: repo, database, CI/CD, and multi-developer Claude Code setup

- **Status:** Proposed (operator review pending)
- **Date:** 2026-08-16
- **Deciders:** Operator + Claude Code (adversarial research workflow: 3 stack advocates → weighted judge → 2 skeptic verification agents; 10 agents, web-verified against primary sources)

## Context

MA is a new, standalone app for a maintenance-service business line: maintenance contracts, repair/service jobs, field technicians (own staff + subcontractor), scheduling, billing, and a customer issue-reporting portal. Thai-language UI for staff; LINE Login + LINE OA messaging are the company norm; mobile-first PWA use in the field.

It is deliberately a **separate repo and separate database** from the existing construction-ops app (`vap-ops/prc-ops`). The client/subcontractor list is short and will be imported once from Google Sheets — no runtime coupling between the apps.

The development team is **three humans, all non-developers, each driving their own Claude Code instance** against this repo. There is no human code review; quality is enforced by automated gates and AI review. The main operator works from a Windows Server 2019 cloud PC; the other two seats are cloud-first (claude.ai/code).

prc-ops carries two years of operational lessons. A pain-mining pass over its memory corpus found that **roughly 70% of recorded pain traces to two shared singletons: one shared remote dev/test database, and one shared Windows box.**

## First invariant

> **No shared mutable singletons in the development or test path.**

Every design decision below either serves this invariant or inherits a proven prc-ops pattern that does not violate it.

## Decision summary

| Area | Decision | Rejected |
|---|---|---|
| Stack | Next.js App Router + Supabase + Vercel + pnpm + vitest + pgTAP ("same stack, refined") | Neon+Drizzle+tRPC (7.4/10), Convex (7.0/10) — see §Alternatives |
| Repo | `vap-ops/ma-ops`, private, GitHub Team plan, flat single-level layout | New org; nested layout |
| Schema channel | Numbered migrations, immutable once merged, CI-enforced | Declarative-diff as the channel (diff engine mishandles `alter policy`/grants/triggers) |
| Test DB | Ephemeral `supabase start` on the CI runner, per PR, replay-from-zero | Shared remote dev/test DB (prc-ops's largest pain class) |
| Prod schema | Applied only by GitHub Actions on merge, sequenced with deploy | Interactive `db:push` from sessions |
| Merge path | Rulesets: PR + required checks + require-up-to-date + squash-only + auto-merge; **no bypass actors** | GitHub merge queue (unavailable on private Team repos; negative value at 3 devs) |
| Coordination | Draft PR = work claim; Issues = backlog; `schema` label = single schema lane; domain ownership map | LANES.md-style local files (single-writer, single-machine) |
| Review | claude-code-action AI review on every PR (advisory → required) + session-side reviewer subagent | Human review (no humans qualified); Managed Code Review (~$15–25/PR, revisit later) |
| Teammate env | claude.ai/code cloud sessions (zero local toolchain) | Local Docker per machine (IT-support burden on non-devs) |
| Auth | Supabase `custom:line` OIDC (GA June 2026), `email_optional: true`, access-token hook carrying role | Magic-link bridge (the prc-ops workaround; obsolete) |
| Secrets | `service_role`/prod-DB/LINE secret only in Vercel + Actions; teammates hold anon-tier only | Secrets on teammate machines / via chat |

## S1 — Repo + toolchain

`vap-ops/ma-ops`, private. pnpm, Next.js App Router (current stable at bootstrap), TypeScript strict, Tailwind, vitest, pgTAP.

From commit #1 (each kills a prc-ops retrofit):
- `.gitattributes` with `* text=auto eol=lf` — no CRLF phantom diffs.
- Format-check as a required CI job — drift can never accumulate (prc-ops `main` carries real prettier drift today; `pnpm format` rewrites ~55 unrelated files).
- NUL/text-integrity guard (a NUL byte once made a source file render as an empty "binary" diff).
- Secret-shape grep in CI (GHAS push protection costs extra on private repos; the grep + the secrets architecture in S6 is proportionate).

Canonical executable ops surface — scripts, not prose memory:
- `scripts/verify` — runs the whole local gate, writes its own log, prints structured `EXITCODE=`, never piped (pipes have swallowed red suites into committed "greens").
- `scripts/pr-status` — the one truthful PR/merge probe (GraphQL `mergeQueueEntry`-class lessons baked in).
- `pnpm doctor` — onboarding prerequisite checker, ✅/❌ per item.

Layout:

```
ma-ops/
├── CLAUDE.md              # doctrine, ≤200 lines
├── REVIEW.md              # AI-review severity calibration
├── .claude/
│   ├── settings.json      # shared permissions + hooks (exec-form, ${CLAUDE_PROJECT_DIR})
│   ├── skills/            # ma-doctrine, ship-unit
│   ├── agents/            # code-reviewer.md
│   ├── rules/             # path-scoped: database.md, ui.md
│   └── hooks/             # guard scripts (portable node)
├── docs/
│   ├── adr/               # this file is 0001
│   ├── specs/             # feature specs
│   ├── OWNERSHIP.md       # domain map (3 humans ↔ 3 domains)
│   ├── ONBOARDING.md      # CC-executable bring-up
│   └── automations.md     # every automated behavior, SSOT
├── src/
├── supabase/
│   ├── migrations/
│   ├── tests/             # pgTAP
│   └── seed.sql           # hermetic fixtures only
└── scripts/{verify, pr-status, doctor}
```

## S2 — Database

New Supabase project (same org as prc-ops; one bill). **Numbered migrations are the only schema channel.** Declarative schema files were evaluated and rejected as the channel: Supabase's diff engine mishandles `alter policy`, grants, and triggers — exactly the objects an RLS-heavy app changes most.

CI-enforced migration rules (enforced, not remembered):
- Immutable once merged — a lint blocks any edit to a migration file that has reached `main` (editing an applied migration silently no-ops on the applied DB but replays elsewhere; three prc-ops incidents).
- Versions unique + monotonic (two prc-ops lanes once shipped the same timestamp, both merged green).
- **Replay-from-zero on every PR** against the runner's ephemeral DB — iteration is free, wedges surface pre-merge.

**Migration 0001 = posture lockdown:**
- Revoke default privileges from `PUBLIC`, `anon`, `authenticated` (a new table is born public; prc-ops learned this as a live finding).
- Closed-world pgTAP check: every table has RLS enabled + a pinned 4-posture matrix. This is what makes RLS genuinely fail-closed — the skeptic pass confirmed RLS is off-by-default per table, so the closed-world check is load-bearing, not decoration.
- UTC storage + ONE civil-date helper (SQL + TS); lint bans bare `current_date` / `new Date().toISOString().slice()` (prc-ops's daily 23:00–24:00 UTC test-eject window class).
- Injectable/frozen clock in DB tests.
- `seq bigint identity` convention on append-only tables (ordering ties on `created_at` produce flaky assertions).

Auth: Supabase **custom OIDC provider for LINE Login** — GA since June 2026, no beta label, `email_optional: true` supported (most Thai LINE accounts expose no email). Custom access-token hook carries the role claim. Roles (initial set, to be finalized in the domain-model spec): `super_admin`, `office`, `auditor`, `technician`, `customer` — plus a role the operator referred to as "cc", to be defined before the roles migration ships. A `dev-preview` magic-link super-admin account exists on preview builds only.

RLS + security-definer RPC conventions carry over from prc-ops verbatim (the guard library was verified stack-generic: `run-pgtap.ts`, posture-pin patterns, migration doctrine).

`database.types.ts` is generated only from a branch-local ephemeral stack (= main + your branch's migrations), never from shared live state.

No Supabase image transformations — pre-generated thumbnails from day 1 (Pro plan's origin-image quota is an org-level trap prc-ops hit).

## S3 — CI/CD + merge path

Required checks (parallel, target <10 min total): `typecheck` · `lint+format` · `vitest` · `db` (ephemeral `supabase start`: replay-from-zero + pgTAP + posture pins) · `build` · `repo-lint` (guards as cached lint rules over **derived** registries — fs-walking vitest guards over hand-lists were prc-ops's #1 flake source) · `ai-review` (claude-code-action; advisory at first, promoted to required once its false-positive rate is measured).

The `db` check running on **every PR ref, in parallel**, is the structural fix for prc-ops's worst class: schema breaks that surfaced only as merge-queue ejections on a ref no PR check displays. Here a schema break reds only its own PR.

Ruleset on `main` (Rulesets, not classic branch protection):
- Require PR; **required approvals: 0** — the gate is CI, not eyes.
- Require status checks + **require branch up-to-date before merging** — the merge-queue substitute: the second PR must update and re-run CI on the real merged combination.
- Squash-only, linear history, block force-push, restrict deletions.
- **No bypass actors, admins included** — one merge path for all PRs. (prc-ops's by-design-failing "danger-path guard" check forced routine admin bypasses; twice a break landed armed and silent.)
- Repo allows auto-merge; ship ritual = `gh pr merge --auto --squash`.

**No merge queue.** Verified: GitHub merge queue requires a public org repo or GitHub Enterprise Cloud — not available on a private Team-plan repo. Even if available: at 3 committers the serialization pressure is low and prc-ops demonstrates the cost side (cross-lane ejections, lying probes, a whole operations playbook). Escape hatches if PR volume ever forces it: Mergify, or Enterprise Cloud. Do not pre-buy.

Danger path: a CI check fails on `DROP|TRUNCATE|ALTER .* DROP` patterns in migrations until a `danger-approved` label is present; only the operator applies that label. Destructive change = 🔔 to operator, in-band.

Prod: Vercel deploys `main` on merge; migrations applied **only by GitHub Actions** using a CI-held Supabase access token, sequenced before app deploy. No interactive push path to prod exists; teammates never hold prod DB credentials. Standing rule: **main red → revert first, debug second** (squash commits make revert one command).

Previews: Vercel preview per PR. Supabase preview branches are **opportunistic, never required infra** — the skeptic pass refuted "production-ready": Supabase stages Branching as Beta (Aug 2026) and per-branch secret drift is officially documented. Preview-app DB wiring is an open item (§Open items).

## S4 — Multi-developer coordination

Coordination state lives **on the server (GitHub), never in a local file**. prc-ops's LANES.md / MEMORY.md single-writer clobber class cannot exist at 3 machines by design.

- **Claim = draft PR** opened at session start: title `[domain] unit description`, body lists directories/tables touched. Visible to every session via `gh pr list`; expires automatically on merge/close.
- **Issues = backlog** (menu, not queue); domain labels give a one-glance lane map.
- **`schema` label = the single schema lane.** One schema PR in flight repo-wide (plan lanes by the shared singleton — for 3 devs on one Postgres, the schema is the singleton). An advisory CI check warns when two open PRs touch `supabase/migrations/`.
- `docs/OWNERSHIP.md`: three domain homes — (A) contracts + billing, (B) jobs + scheduling + technicians, (C) customer portal + LINE messaging. Cross-domain work is announced in the team LINE group first. At 3 people, a sentence in chat beats tooling.
- **Auto-push on every commit** (hook) — "work exists on one disk" is structurally impossible (prc-ops once had 7 commits + 5 dirty files sit unpushed on one disk for 9 days).
- One CC session per human at a time.

Session-start ritual (in CLAUDE.md, executed by every CC session):
1. `git fetch && git status` — clean tree or stop.
2. `gh pr list --state open` — read others' claims; do not start work overlapping an open draft PR's scope.
3. New work → branch + push + `gh pr create --draft`.
4. Writing a migration → check `gh pr list --label schema` first.

## S5 — Claude Code architecture (3 seats)

Everything shared is **in-repo**, inherited via git, identical on all seats:

- `CLAUDE.md` (≤200 lines): session-start ritual, ship ritual, never push `main`, never edit merged migrations, one-schema-lane, TDD default, evidence-before-assertion digest (claim nothing done without command output), danger path = ask the operator, Thai UI terms from the labels SSOT.
- `.claude/settings.json`: shared permission allowlist (identical prompts on all seats) + **hooks as hard guardrails** — PreToolUse blocks on `git push` to `main` and on edits to already-merged files under `supabase/migrations/`. Hooks use exec form with `${CLAUDE_PROJECT_DIR}` and Node scripts → portable across Windows/Mac/Linux/cloud.
- `.claude/skills/`: `ma-doctrine` (the operating checklist), `ship-unit` (the gated build-and-ship loop, ported from prc-ops).
- `.claude/agents/code-reviewer.md`: the session-side fresh-eyes reviewer (prc-ops Gate 4), used before every ship in addition to the CI review.
- `.claude/rules/`: path-scoped rules — `database.md` loads only when touching `supabase/**`, `ui.md` for `src/**` UI work.
- `REVIEW.md`: severity calibration for the CI AI review (blocking = logic, security, RLS scope, missing tests; nit = style).

Per-machine quirks (paths, shell, the Windows Thai-text-via-Edit-tool rule) go in each machine's gitignored `CLAUDE.local.md` — one machine's quirks never pollute shared doctrine.

**Knowledge policy:** shared project knowledge (lessons, decisions, gotchas) lives in-repo — `docs/adr/`, `docs/automations.md`, rule files — where all three CCs load it. Personal auto-memory remains personal per seat. The operator's prc-ops-style memory system continues for the operator's own workflow, but MA project facts belong in the repo.

Teammates #2/#3 run **claude.ai/code cloud sessions**: org cloud-environment config with a setup script (`pnpm install`; env = anon key + URLs only), GitHub App auth, `/teleport` to a terminal as escape hatch. Ephemeral Linux VM per session — the entire Windows quirk catalog never applies to them. Verification of user-facing work happens on per-PR Vercel preview URLs, not local dev servers.

`docs/ONBOARDING.md` is written to be **executed by a new teammate's CC session** ("paste this into Claude Code"), ending with `pnpm doctor` printing ✅/❌ evidence.

Human protocol: when CC is stuck or claims something scary — ask in the team LINE group with the transcript; never approve a CC request you don't understand.

## S6 — LINE + secrets

- New LINE **Login channel** + new **OA** dedicated to MA. Never reuse or repoint prc-ops channels (`line-channels-topology` law: prod Login channel binds to prod URL only).
- LINE Login requires exact pre-registered callback URLs → **LINE auth cannot work on per-PR preview URLs.** Previews use the `dev-preview` magic-link account. A separate dev LINE channel arrives with staging.
- Staging: **deferred** until real customer billing flows exist. Pre-revenue, per-PR previews + fast revert cover the need; an unattended staging just drifts.
- Secrets: `service_role`, prod DB password, prod LINE channel secret → Vercel env + GitHub Actions secrets only; no human machine holds them. Teammate `.env.local` = anon/publishable key + URLs (low sensitivity). Distribution via Doppler (free tier, 3 seats — re-verify at signup) or operator sets directly. Secrets never travel through LINE/chat.

## S7 — Deliberately not inherited from prc-ops

| Dropped | Replaced by |
|---|---|
| Merge queue + probe playbook | Rulesets + require-up-to-date + auto-merge |
| `known-red.json` quarantine machinery | Hermetic ephemeral test DBs; red is fixed or reverted now |
| LANES.md / local coordination files | Draft-PR claims + Issues (server-side, atomic) |
| `ship-pr.sh` bespoke PAT push path | Credentials designed day 1: `gh` + credential helper; plain `git push` |
| Shared hot tracker file (append ping-pong) | Changesets pattern: one file per change, CI aggregates |
| Hand-list registries + fs-walking guard tests | Derived registries + cached repo-lint |
| Supabase image transformations | Pre-generated thumbnails |
| Magic-link LINE auth bridge | `custom:line` OIDC (GA) |
| Shared dev/test DB; interactive `db:push` | Ephemeral CI DBs; CI-only prod apply |
| Double-nested repo directory | Flat `D:\claude\projects\ma-ops` |

## Alternatives considered (stack)

Adversarial evaluation, weighted for this buyer (build capacity is entirely AI-authored; no human review; foundation-first):

- **B — Neon + Drizzle + tRPC + better-auth (7.4/10).** Best DB branching (copy-on-write with data) and best pure-testing ergonomics. Rejected: app-layer authorization fails **open** on one forgotten guard (its own advocate conceded), and ~55% of the prc-ops quality-system library is stranded. **Revisit if:** Supabase GitHub-mode branching proves too flaky for 3 seats AND the team budgets 1–2 months to rebuild guards AND a mechanically airtight protected-base lint exists; or if the custom OIDC provider regresses.
- **C — Convex (7.0/10).** Most managed, best reactivity, per-dev cloud deployments. Rejected: thinnest AI training corpus (its advocate conceded orders of magnitude vs Postgres/Next), same fail-open authz shape, worst lesson transfer (~35%), paradigm lock-in, and the operator's probe-by-SQL working habit dies.

Load-bearing verified facts (skeptic pass, primary sources, Aug 2026):
1. Supabase custom OAuth/OIDC providers: **GA**, `email_optional` configurable; LINE serves a live OIDC discovery document. ([docs](https://supabase.com/docs/guides/auth/custom-oauth-providers), [feature page](https://supabase.com/features/custom-oidc-providers))
2. prc-ops guard library is stack-generic → transfers near-verbatim (verified against the repo files, not assumed).
3. Docker-free workflow is official: Actions runner runs `supabase start` + `supabase test db`; MCP covers interactive DB work. ([CI testing docs](https://supabase.com/docs/guides/deployment/ci/testing))
4. Supabase Branching is **Beta** (not GA); per-branch secret drift documented → previews stay opportunistic. ([feature page](https://supabase.com/features/branching), [branching docs](https://supabase.com/docs/guides/deployment/branching))
5. GitHub merge queue is unavailable on private Team-plan repos. ([community #131130](https://github.com/orgs/community/discussions/131130))

## Open items

1. **Domain model** (contracts / jobs / assets / visits shape, flexibility for future business lines) — own brainstorm + spec, next.
2. **Role list finalization** — the operator's list included a role written as "cc"; define it (and confirm the full enum) before the roles migration ships.
3. **Preview-app DB wiring** — per-PR Vercel preview needs a database: Supabase preview branch when it works (Beta), else a shared staging-lite branch DB (acceptable: hermetic tests never touch it). Decide at bootstrap when the integration is exercised.
4. **Client/subcon import** — one-time Google Sheets import script; shape decided with the domain model.
5. **Doppler free-tier seat count** — most volatile external fact here; re-verify at signup.
6. **GitHub Team plan** — confirm the org is on Team (Rulesets' required-status-checks + auto-merge on a private repo need it).

## Bootstrap order (sketch — full plan via writing-plans)

1. Scaffold: Next.js + pnpm + TS strict + Tailwind + vitest; `.gitattributes`; CI skeleton with all required checks red-until-real.
2. Supabase project + migration 0001 posture lockdown + pgTAP harness + ephemeral-DB CI job.
3. Rulesets + auto-merge + claude-code-action review + danger-path check.
4. `.claude/` doctrine set + `CLAUDE.md` + `REVIEW.md` + `scripts/{verify,pr-status,doctor}`.
5. Auth: `custom:line` + access-token hook + dev-preview account.
6. `docs/ONBOARDING.md` + cloud-environment config; onboard teammate #2 as the runbook's first test.
7. Domain-model spec → first feature units.
