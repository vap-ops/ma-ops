# MA — Claude Code doctrine (seed)

Pre-bootstrap seed. The full doctrine ships with bootstrap step 4 (ADR 0001 §Bootstrap). Until then, the rules below are already binding.

## What this project is

Read [docs/adr/0001-founding-architecture.md](docs/adr/0001-founding-architecture.md) before proposing or building anything. It is the single source of truth for stack, CI gates, merge path, coordination, and the Claude Code setup. Do not re-litigate decided items casually; open a new ADR to supersede.

## Binding rules (from day 0)

- **Session start:** `git fetch && git status` (clean tree or stop) → `gh pr list --state open` (read others' claims; do not overlap an open draft PR's scope) → new work gets a branch + push + draft PR (`[domain] unit description`, body lists directories/tables touched).
- **Never push `main` directly.** Ship = PR + green checks + `gh pr merge --auto --squash`.
- **Never edit a migration file that has reached `main`** — always a new file.
- **One schema PR in flight repo-wide** (`schema` label; check `gh pr list --label schema` first).
- **Evidence before assertion:** claim nothing as done/passing without the command output in front of you.
- **Destructive change (DROP/TRUNCATE/data loss) = ask the operator first.** CI will block it without the `danger-approved` label.
- **Push every commit** — work must never exist on one disk only.
- Thai text is written via the Edit/Write tools only (never through PowerShell heredocs).

## Layout

Flat repo at `D:\claude\projects\ma-ops` on the operator's machine (no nesting). Structure per ADR 0001 §S1.
