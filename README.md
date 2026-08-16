# ma-ops

Maintenance-service operations app (MA): maintenance contracts, repair/service jobs, field technicians, scheduling, billing, and a customer issue-reporting portal. Thai-first UI, LINE Login, mobile-first PWA.

Sibling of [`vap-ops/prc-ops`](https://github.com/vap-ops/prc-ops) (construction ops) — deliberately separate repo and database; shared lessons, not shared runtime.

## Status

Pre-bootstrap. The founding architecture is decided and recorded in
[docs/adr/0001-founding-architecture.md](docs/adr/0001-founding-architecture.md) — read that first; it defines the stack, CI gates, merge path, multi-developer coordination, and the Claude Code setup for all three seats.

## Development

Development is done by Claude Code driven by non-developer operators. Every session starts by reading `CLAUDE.md` (auto-loaded) and following its session-start ritual. Quality is enforced by CI gates and AI review, not human review — see ADR 0001 §S3.
