# agentic-workflows

A collection of AI agent skills and the workflows that compose them.

## Skills

- [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave) — Parallel agent coordination pattern. A scout maps file ownership and interface contracts; parallel agents execute in waves with build/test gates between them.
- [ai-cold-start-audit](https://github.com/blackwell-systems/ai-cold-start-audit) — Cold-start UX auditing. AI agents simulate new users in containerized sandboxes and produce severity-tiered findings reports.

## Workflows

- [Audit-Fix-Verify](workflows/audit-fix-verify/) — Use cold-start-audit to find issues, SAW to fix them in parallel, re-audit to confirm resolution.

## Philosophy

These tools were designed independently but compose because they share a common artifact format — structured, severity-tiered markdown with exact reproduction steps. The audit's output is the scout's input. The loop is repeatable as a release gate.
