# agentic-workflows

AI agents compose. A cold-start audit finds what's broken. A parallel fix wave repairs it. A re-audit confirms no regressions were introduced. This repo documents the skills and workflows that make that loop repeatable — and a reference implementation with hard numbers from six complete cycles.

---

## Skills

### [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave) (SAW)

Parallel agent coordination protocol. Before any agent starts, a scout analyzes the codebase to produce file ownership maps and interface contracts. Agents then execute in waves with build and test gates between them. The scout's pre-implementation check runs first — it filters items that are already implemented before agents launch, eliminating wasted work.

Key guarantees: agents work concurrently without file conflicts; waves enforce dependency order; tests must pass before the next wave starts.

### [agentic-cold-start-audit](https://github.com/blackwell-systems/agentic-cold-start-audit)

Cold-start UX auditing. Turns AI's lack of context into a feature: agents simulate new users in sandboxed environments (Docker container, local env var isolation, or git worktree) and produce severity-tiered findings reports with exact reproduction steps.

The three sandbox modes cover the full range of tool types: container for destructive operations, local for tools with redirectable state, worktree for directory-scoped tools. Output is a structured findings report — severity tier, affected area, reproduction steps — formatted to feed directly into SAW.

---

## Workflows

### [Audit-Fix-Verify](workflows/audit-fix-verify/)

1. Run `cold-start-audit` against your tool. Get a severity-tiered findings report with exact reproduction steps.
2. Feed the findings report to `SAW scout`. The scout reads audit findings alongside the codebase to produce file ownership, interface contracts, and a wave structure. Run `/saw wave --auto` to execute.
3. Re-run `cold-start-audit` against the fixed build. Compare against the original report: resolved, regressed, surviving, new.

The two tools compose because their artifact formats are compatible. The audit produces severity-tiered findings grouped by area; the SAW scout reads those tiers and groupings directly to decide wave order and agent assignments. No manual translation step.

The deeper insight: the audit's lack-of-context is the UX signal — an agent that can't figure out what to do next is faithfully simulating a new user hitting the same wall. SAW's pre-implementation check then filters already-fixed items before agents run, so each round only works on what's actually broken.

See [the full workflow doc](workflows/audit-fix-verify/) for the complete loop, sandbox mode selection, severity-to-wave mapping, and what the re-audit output means.

---

## Reference Implementation

[brewprune](https://github.com/blackwell-systems/brewprune) — a Homebrew package usage tracker and removal advisor — is the primary proof-of-concept for both tools and the Audit-Fix-Verify workflow. Six complete audit→fix cycles.

**Aggregate numbers:**
- 52 agents across 6 rounds
- 118 findings fixed across 79 files

**Round 2:** 11 agents, single wave, +4,021 lines added, all tests green.

**Round 5:** The pre-implementation check eliminated 12 of 24 findings before any agent launched — they had already been implemented. Eight minutes saved per run, work that would otherwise have been duplicated.

**Round 6:** 8 agents, single wave. Wall-clock: 13 minutes. Estimated sequential: 64 minutes. **80% faster than sequential.**

**Notable incidents:**

*Round 4 — scout refused to write its own output.* The scout's role was described as "read-only investigation." The scout interpreted that literally and declined to write the IMPL doc it was supposed to produce. Fixed with explicit permission language: the scout's investigation is read-only, but producing the IMPL doc is its required output. One-line prompt change; logged in the protocol.

*Round 5 — self-healing worktree.* Two agents independently hit a state where `cd` had put them in a broken directory. Both triggered the cd-then-verify recovery path built into the skill, recovered without human intervention, and documented their own recovery in their completion reports. The protocol held under real failure conditions.

The brewprune `docs/` directory contains every IMPL doc and audit report from all six rounds. The full progression — what broke, what was fixed, what regressed, what the loop caught — is documented in [brewprune-example.md](workflows/audit-fix-verify/brewprune-example.md).

---

## Blog

Three-part series on scout-and-wave:

1. [A Coordination Pattern for Parallel AI Agents](https://blog.blackwell-systems.com/posts/scout-and-wave/) — The pattern, the scout's role, and a worked example of wave structure and file ownership.
2. [What Dogfooding Taught Us](https://blog.blackwell-systems.com/posts/scout-and-wave-part2/) — The audit-fix-audit loop in practice: overhead measurement, Quick mode for small scopes, and the bootstrap problem (using SAW to build SAW).
3. [Five Failures, Five Fixes](https://blog.blackwell-systems.com/posts/scout-and-wave-part3/) — How the skill file decomposed from a monolith into composable pieces, and the scout prompt's bug tracker.

---

## How They Compose

Cold-start-audit and scout-and-wave solve adjacent problems, and they're designed to hand off to each other cleanly.

The audit produces findings in a format the scout can consume directly: severity tiers map to wave priority (UX-critical → Wave 0 or 1, UX-improvement → Wave 1 or 2, UX-polish → last), and area groupings map to agent assignments. The scout doesn't need a reformatted brief — it reads the audit report as-is.

The more important connection is conceptual. The audit's signal is the places where an agent with no context gets stuck or confused. That's not a limitation of the audit methodology — it's the whole point. A new user hitting the same wall is the UX failure you're trying to find. The audit surfaces those failures with exact reproduction steps. SAW then fixes them in parallel with verified coordination.

The pre-implementation check closes the loop on round-over-round efficiency: each re-audit produces a smaller findings set as the tool improves, and SAW filters whatever was already fixed before agents run. The loop gets faster as the tool gets better.
