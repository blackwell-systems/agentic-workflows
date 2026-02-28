# agentic-workflows

A collection of AI agent skills and the workflows that compose them.

## Skills

- [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave) — Parallel agent coordination pattern. A scout maps file ownership and interface contracts; parallel agents execute in waves with build/test gates between them.
- [agentic-cold-start-audit](https://github.com/blackwell-systems/agentic-cold-start-audit) — Cold-start UX auditing. AI agents simulate new users in containerized sandboxes and produce severity-tiered findings reports.

## Workflows

- [Audit-Fix-Verify](workflows/audit-fix-verify/) — Use cold-start-audit to find issues, SAW to fix them in parallel, re-audit to confirm resolution.
