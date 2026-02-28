# brewprune — Audit-Fix-Verify Reference Run

A complete documented run of the audit-fix-verify workflow applied to [brewprune](https://github.com/blackwell-systems/brewprune), a Homebrew package usage tracker and removal advisor.

## Context

- **Tool:** brewprune — a Homebrew package usage tracker and removal advisor
- **First audit:** February 2026, cold-start agent in `bp-sandbox` Docker container
- **First findings:** 21 issues — 5 UX-critical, 9 UX-improvement, 7 UX-polish
- **Fix method:** Scout-and-Wave, 3 waves, 5 agents
- **Re-audit:** Same container rebuilt from fixed source
- **Re-audit findings:** 18 issues — 5 UX-critical, 8 UX-improvement, 5 UX-polish

## Wave structure produced by SAW scout

```
Wave 0 ── Agent 0: confidence.go (score logic inversion — correctness prerequisite)
              │
Wave 1 ── Agent A: scan.go, quickstart.go, watch.go, status.go
          Agent B: explain.go, doctor.go, stats.go, undo.go
              │
Wave 2 ── Agent C: output/table.go, output/progress.go
          Agent D: unused.go, remove.go, root.go
```

## What the re-audit found

**Resolved from original criticals:**

- Score logic inverted (Wave 0 fix — `computeUsageScore` now correctly awards 0 pts for recently-used packages)
- `unused` shows nothing with no data (Wave 2 — `showRiskyImplicit` shows all packages with warning banner when no usage events exist)
- Spinner garbage in non-TTY output (Wave 2 — `writerIsTTY()` helper, non-TTY path skips animation)
- ANSI color leaks to non-TTY (Wave 2 — `IsColorEnabled()` checks `NO_COLOR` + `isatty`)

**Regression introduced by fixes:**

- `unused --tier risky` shows `✗ keep` for all rows — Wave 2 changed the risky tier label from "RISKY" to "✗ keep" (correct in the default mixed-tier view). When a user explicitly filters `--tier risky`, every row saying "keep" contradicts why they ran the command. Context-dependent label, single implementation.

**New findings not in original audit:**

- `doctor` output says "Fix: add shim directory to PATH" — the word "Fix:" implies a `--fix` flag that does not exist. Running `brewprune doctor --fix` errors.

**Surviving exit code issues:**

- `explain nonexistent` exits 0 on package-not-found
- `undo latest` exits 0 when no snapshots exist

## What this illustrates

The re-audit caught a regression (the `✗ keep` label) that code review and tests missed — tests verified behavior, not rendered label strings in filtered contexts. It also found a new issue (`doctor --fix`) that the original audit didn't probe. The total count dropped from 21 to 18, but more importantly the original four non-exit-code criticals were resolved and one regression was caught before release.

Wall-clock time for the full fix cycle: ~29 minutes (scout 7m + Wave 0 5m + Wave 1 11m parallel + Wave 2 7m parallel). Estimated sequential: 35-40 minutes.
