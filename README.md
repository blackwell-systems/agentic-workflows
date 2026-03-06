# agentic-workflows

[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)

AI agents compose. A cold-start audit finds what's broken. A parallel fix wave repairs it. A re-audit confirms no regressions were introduced. This repo documents the skills and workflows that make that loop repeatable.

Each skill is a protocol-bearing unit: explicit inputs, explicit outputs, defined behavior at its boundaries. The LLM executes the steps; the protocol enforces the interface. Handoffs become as deterministic as function calls, even though both sides run on natural language instructions.

For multi-agent work, the same principle applies at the coordination layer. SAW's IMPL doc is a shared contract all agents write to and read from. Parallel execution is safe not because agents trust each other, but because they're all operating against the same structure.

Skills are independently useful, but more powerful in combination.

---

## Skills

### [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave) (SAW)

Parallel agent coordination protocol. Before any agent starts, a scout analyzes the codebase to produce file ownership maps and interface contracts. Agents then execute in waves with build and test gates between them. The scout's pre-implementation check runs first, filtering items that are already implemented before agents launch and eliminating wasted work.

Key guarantees: agents work concurrently without file conflicts; waves enforce dependency order; tests must pass before the next wave starts.

<details>
<summary>Execution flow diagram</summary>

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/diagrams/saw-scout-wave-dark.svg">
  <img src="docs/diagrams/saw-scout-wave-light.svg" alt="SAW scout + wave execution flow" style="max-width: 100%;">
</picture>

</details>

### [github-release-engineer](https://github.com/blackwell-systems/github-release-engineer)

Automated GitHub release pipeline. Runs pre-flight checks, detects or confirms the version, promotes the changelog entry, creates an annotated tag, pushes it, watches CI to completion, waits for release artifacts, and verifies the published release. Each step gates the next; failures surface immediately with enough context to fix them.

On completion, the release engineer hands off to companion skills (e.g. `homebrew-formula-updater`) with the full release context: version, tag SHA, release URL, and asset list with checksums already resolved.

### [homebrew-formula-updater](https://github.com/blackwell-systems/homebrew-formula-updater)

Updates a Homebrew tap formula to a new release. Locates the tap on disk (or clones it), verifies the working tree is clean, resolves checksums from the GitHub release asset, updates the formula's version, URLs, and SHA256 fields only, shows a diff for confirmation, and commits and pushes.

Designed to be invoked by `github-release-engineer` at the end of a release, or standalone. Checksum sourcing priority: checksum file attached to the release → checksums passed by the release engineer → ask user.

### [agentic-cold-start-audit](https://github.com/blackwell-systems/agentic-cold-start-audit)

Cold-start UX auditing. Turns AI's lack of context into a feature: agents simulate new users in sandboxed environments (Docker container, local env var isolation, or git worktree) and produce severity-tiered findings reports with exact reproduction steps.

The three sandbox modes cover the full range of tool types: container for destructive operations, local for tools with redirectable state, worktree for directory-scoped tools. Output is a structured findings report (severity tier, affected area, reproduction steps) formatted to feed directly into SAW.

### [dockerfile-sandbox-gen](https://github.com/blackwell-systems/dockerfile-sandbox-gen)

Dockerfile generator for sandboxed tool testing. Detects project language (Go, Rust, Python, Node.js), selects the appropriate template, and generates a multi-stage Dockerfile with configurable tool names and build commands. Outputs to `Dockerfile.sandbox` by convention, which cold-start-audit's `--dockerfile` parameter can consume. Independently useful for any workflow requiring containerized tool builds.

Key features: automatic language detection, multi-stage builds (compile → run separation), template validation, and variable substitution. Composes with cold-start-audit via file convention, not hard dependency.

---

## Workflows

### [Release-Publish](workflows/release-publish/)

1. Run `/release` from the project repo. The release engineer handles pre-flight, changelog promotion, tagging, CI monitoring, and artifact verification.
2. On success, the release engineer automatically invokes `/homebrew-formula-updater` with the release context. The formula updater reads checksums from the attached `checksums.txt`, updates the tap formula, and pushes.
3. Run `brew update && brew upgrade <tool>` to confirm the published formula installs cleanly.

The two skills compose because the release engineer's completion report carries exactly what the formula updater needs: version, tag SHA, release URL, and checksums, with no manual translation step. The formula updater's checksum resolution falls back gracefully if the release engineer didn't pass them.

See [the full workflow doc](workflows/release-publish/) for setup, configuration, and what to do when CI fails mid-release.

### [Audit-Fix-Verify](workflows/audit-fix-verify/)

1. Run `cold-start-audit` against your tool. Get a severity-tiered findings report with exact reproduction steps.
2. Feed the findings report to `SAW scout`. The scout reads audit findings alongside the codebase to produce file ownership, interface contracts, and a wave structure. Run `/saw wave --auto` to execute.
3. Re-run `cold-start-audit` against the fixed build. Compare against the original report: resolved, regressed, surviving, new.

The two tools compose because their artifact formats are compatible. The audit produces severity-tiered findings grouped by area; the SAW scout reads those tiers and groupings directly to decide wave order and agent assignments. No manual translation step.

The deeper insight: the audit's lack-of-context is the UX signal. An agent that can't figure out what to do next is faithfully simulating a new user hitting the same wall. SAW's pre-implementation check then filters already-fixed items before agents run, so each round only works on what's actually broken.

See [the full workflow doc](workflows/audit-fix-verify/) for the complete loop, sandbox mode selection, severity-to-wave mapping, and what the re-audit output means.

---

## How They Compose

Cold-start-audit and scout-and-wave solve adjacent problems, and they're designed to hand off to each other cleanly.

The audit produces findings in a format the scout can consume directly: severity tiers map to wave priority (UX-critical → Wave 0 or 1, UX-improvement → Wave 1 or 2, UX-polish → last), and area groupings map to agent assignments. The scout doesn't need a reformatted brief. It reads the audit report as-is.

The more important connection is conceptual. The audit's signal is the places where an agent with no context gets stuck or confused. That's not a limitation of the audit methodology. It's the whole point. A new user hitting the same wall is the UX failure you're trying to find. The audit surfaces those failures with exact reproduction steps. SAW then fixes them in parallel with verified coordination.

The pre-implementation check closes the loop on round-over-round efficiency: each re-audit produces a smaller findings set as the tool improves, and SAW filters whatever was already fixed before agents run. The loop gets faster as the tool gets better.

---

## Blog

Three-part series on scout-and-wave:

1. [A Coordination Pattern for Parallel AI Agents](https://blog.blackwell-systems.com/posts/scout-and-wave/): The pattern, the scout's role, and a worked example of wave structure and file ownership.
2. [What Dogfooding Taught Us](https://blog.blackwell-systems.com/posts/scout-and-wave-part2/): The audit-fix-audit loop in practice: overhead measurement, Quick mode for small scopes, and the bootstrap problem (using SAW to build SAW).
3. [Five Failures, Five Fixes](https://blog.blackwell-systems.com/posts/scout-and-wave-part3/): How the skill file decomposed from a monolith into composable pieces, and the scout prompt's bug tracker.
