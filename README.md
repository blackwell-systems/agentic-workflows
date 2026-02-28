# agentic-workflows

AI agents compose. A cold-start audit finds what's broken. A parallel fix wave repairs it. A re-audit confirms the regression didn't introduce new problems. This repo documents the skills and workflows that make that loop repeatable.

## Skills

- [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave) — Parallel agent coordination protocol. A scout maps file ownership and interface contracts before any agent starts; parallel agents execute in waves with build and test gates between them. Guarantees agents can work concurrently without conflicts.
- [agentic-cold-start-audit](https://github.com/blackwell-systems/agentic-cold-start-audit) — Cold-start UX auditing. Turns AI's lack of context into a feature: agents simulate new users in containerized sandboxes and produce severity-tiered findings reports with exact reproduction steps.

## Workflows

- [Audit-Fix-Verify](workflows/audit-fix-verify/) — Use cold-start-audit to find issues, SAW to fix them in parallel, re-audit to confirm resolution. The two tools compose because their artifact formats are compatible: audit findings map directly to SAW scout input, no manual translation step.
