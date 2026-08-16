# Port manifest — what comes over from prc-ops, when, and what never does

prc-ops (public: https://github.com/vap-ops/prc-ops · local: `D:\claude\projects\prc-ops\prc-ops`) carries two years of battle-tested artifacts. A 2026-08 verification pass confirmed the ones below are stack-generic. **Rule: no bulk copying.** The plan that needs an artifact ports it, adapts it (paths, project refs, no-merge-queue world), and owns it from then on. A copied file nobody adapted is worse than no file.

## Plan 2 — supabase-foundation

| prc-ops source | What it is | Adaptation |
|---|---|---|
| `scripts/run-pgtap.ts`, `scripts/pgtap-batch.ts`, `scripts/pgtap-report.ts` | pgTAP runner + batch + report | Point at local/ephemeral stack (ma-ops has no shared linked DB); path constants |
| `scripts/gen-db-types.ts` | types generation | Generate from branch-local stack only (ADR §S2) |
| `supabase/tests/` conventions (read several) | posture-pin patterns: born-public revoke checks, `has_table_privilege` allowlists, pinned error messages in `throws_ok` (never `null` msg), fixture-scoped counts, id-anchored rows | Import the CONVENTIONS as day-1 lint + test templates, not the app-specific tests |

## Plan 3 — cc-doctrine-set

| prc-ops source | What it is | Adaptation |
|---|---|---|
| `.claude/skills/ship-unit/SKILL.md` | gated build→ship loop | Remove merge-queue/LANES/ship-pr.sh gates; replace with draft-PR claim + `gh pr merge --auto --squash`; keep gate ORDER (live-form gate-check → TDD RED-first → real-browser verify → fresh reviewer → ship) |
| `.github/workflows/ci.yml` (selectively) | secret-scan job, guard wiring shapes | Cherry-pick jobs; ma-ops CI is already structured differently |
| `docs/automations.md` | every-automation SSOT doc | Copy the structure, empty the content |
| unit-reviewer / fresh-eyes agent prompts | reviewer severity + method | Feed into `.claude/agents/code-reviewer.md` + `REVIEW.md` |

## Plan 4 — auth-line-oidc

| prc-ops source | What it is | Adaptation |
|---|---|---|
| dev-preview login pattern (memory `dev-preview-login`, + its route/seed code) | magic-link super-admin for preview builds | Rebuild small; concept ports, code is app-specific |
| LINE channel topology lesson | prod channel binds prod URL only; never repoint | Doc note in ADR/spec — MA gets its OWN new channel pair |

## Feature-time (port when the first feature needs it)

| prc-ops source | What it is | Trigger |
|---|---|---|
| `src/lib/format.ts` | money/number format SSOT | First money display |
| `src/lib/i18n/labels.ts` PATTERN + its guard test | Thai UI-term SSOT (term used 2+ places → labels file, test-enforced) | First Thai UI copy |
| Field-First design tokens (tailwind theme) | raw-palette-banned design system | First real screen — decide then whether MA shares the design language or diverges |
| `.claude/skills/supersede-pattern/SKILL.md` | append-only edit pattern (supersede/tombstone) | First append-only table (job logs/photos likely) |
| `.claude/skills/bug-fix-flow`, `triage-feedback` | feedback triage + autonomous bug pipeline | When MA has real users + an in-app feedback table; also the blueprint for the `agent` principal's capability list |
| touch-action / keyboard-inset lessons (memories `prc-ops-touch-action-scroll-rows`, `keyboard-occlusion-*`) | mobile scroll + keyboard traps | First scrollable row strip / overlay input |

## Never port (dead classes — ADR §S7)

`ship-pr.sh` · merge-queue playbook + probes · `known-red.json` machinery · LANES.md tooling · progress-tracker hot file · fs-walking guard tests · image-transformation usage · magiclink LINE bridge.
