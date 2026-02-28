# Audit-Fix-Verify

Use [cold-start-audit](https://github.com/blackwell-systems/agentic-cold-start-audit) to find issues, [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave) to fix them in parallel, re-audit to confirm resolution.

## Why

You can't effectively triage and fix a large findings report with a single agent — the scope is too wide and file conflicts make parallel execution unsafe. Scout-and-wave coordinates the fix across multiple agents. But it needs a well-structured problem description to work from. Cold-start-audit provides exactly that.

The two tools compose because their artifact formats are compatible: the audit produces severity-tiered findings with exact reproduction steps and affected areas; the SAW scout reads that alongside the codebase to produce file ownership, interface contracts, and a wave structure. No manual translation step.

## The loop

```
cold-start-audit
      │
      ▼
findings report (severity-tiered, area-grouped, exact repro steps)
      │
      ▼
SAW scout (reads findings + codebase → file ownership + wave structure)
      │
      ▼
parallel agents fix across waves
      │
      ▼
cold-start-audit re-runs against fixed build
      │
      ▼
delta report (resolved / new / regressed)
```

## How to run it

1. **Audit** — `/saw scout "fix UX issues from audit"` after running the cold-start audit. Or: give the SAW scout both the findings report path and the feature description.
2. **Review** — Read the IMPL doc. Verify wave structure makes sense given the severity tiers. UX-critical findings are natural Wave 0 or Wave 1 candidates.
3. **Fix** — `/saw wave --auto` to execute all waves.
4. **Rebuild** — Rebuild the sandbox container from the fixed source.
5. **Re-audit** — `/cold-start-audit run <container> <tool>` against the rebuilt container.
6. **Compare** — Read the new findings against the original. Look for: issues resolved, regressions introduced, new issues surfaced.

## Severity tiers map to wave priority

| Audit tier | SAW wave |
|---|---|
| UX-critical | Wave 0 or Wave 1 — fix before anything else |
| UX-improvement | Wave 1 or Wave 2 — parallelize freely |
| UX-polish | Last wave or batch separately |

## What the re-audit tells you

- **Resolved**: original findings gone from the new report
- **Regressions**: new findings caused by the fixes (the loop catches these; sequential review often misses them)
- **Surviving**: issues that weren't fully addressed
- **New**: pre-existing issues the first audit missed, now surfaced

The re-audit is not optional. Fixes introduce regressions. The loop exists to catch them.

## Reference example

See [brewprune-example.md](brewprune-example.md) for a complete documented run.
