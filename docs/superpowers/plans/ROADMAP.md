# Bootstrap roadmap — plan decomposition

ADR 0001 §Bootstrap, decomposed into plan-sized units. Each unit produces working, independently testable software. **Plans 2–5 are written just-in-time** by the session that executes them (superpowers:writing-plans skill), so version pins and platform facts are probed live, never copied stale from this file.

| # | Plan | Delivers | Depends on |
|---|---|---|---|
| 1 | [2026-08-16-bootstrap-scaffold-ci.md](2026-08-16-bootstrap-scaffold-ci.md) | Next.js app boots; typecheck/lint/test/build CI green; ruleset + auto-merge enforcing on `main`; `scripts/verify` + `pnpm doctor` v0 | — |
| 2 | supabase-foundation (JIT) | Supabase project; migration 0001 posture lockdown (revoke defaults, closed-world RLS pgTAP, civil-date helper, injectable clock); ephemeral-DB `db` CI job (replay-from-zero + pgTAP); `database.types.ts` gen from branch-local stack; migration-immutability + version lints | 1 |
| 3 | cc-doctrine-set (JIT) | Full `CLAUDE.md`; `.claude/settings.json` shared permissions + guard hooks (block push→main, block merged-migration edits); `ma-doctrine` + `ship-unit` skills; `code-reviewer` agent; `.claude/rules/`; `REVIEW.md`; claude-code-action AI-review workflow (advisory); danger-path check + `danger-approved` label | 1 |
| 4 | auth-line-oidc (JIT) | `custom:line` OIDC provider (email_optional), access-token hook with role claim, roles enum (`super_admin/office/auditor/technician/customer`), **`agent` service principal**, `dev-preview` magic-link account on previews | 2 |
| 5 | onboarding-cloud-env (JIT) | `docs/ONBOARDING.md` (CC-executable), `pnpm doctor` full, claude.ai/code cloud-environment config, `docs/OWNERSHIP.md`, Doppler (or operator-set) secret distribution | 1–4 |

Then: domain-model brainstorm (contracts/jobs/assets/visits) → feature specs → normal ship loop.

Rules for JIT plan authors:
- Read ADR 0001 first; it is the SSOT. A plan that contradicts it needs an ADR first.
- Probe live facts (versions, plan quotas, beta status) before pinning them in steps — the ADR's §verified-facts carry Aug-2026 dates and expire.
- Every plan follows superpowers:writing-plans format: exact paths, complete code in steps, TDD, frequent commits.
