# agentic-workflows

[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)
[![Agent Skills](badge-agentskills.svg)](https://agentskills.io)

> [!NOTE]
> All skills in this ecosystem follow the [Agent Skills](https://agentskills.io) open standard - compatible with Claude Code, Cursor, GitHub Copilot, Gemini CLI, and other Agent Skills-compatible tools. Install any skill in your preferred tool using the same directory structure.

AI agents compose through compatible data formats. Skills output structured artifacts that other skills consume. No hard dependencies, no API contracts - just convention-based interop.

Each skill is a protocol-bearing unit: explicit inputs, explicit outputs, defined behavior at its boundaries. The LLM executes the steps; the protocol enforces the interface. Handoffs become as deterministic as function calls, even though both sides run on natural language instructions.

Skills are independently useful, but more powerful in combination.

---

## Composition Philosophy

These skills compose through **compatible data formats**, not hard dependencies. The Unix philosophy for AI workflows: each tool does one thing well, outputs structured artifacts, and remains independently useful.

**Key principles:**

1. **Data contracts over API calls** - Tools agree on artifact formats (IMPL docs, findings reports, release metadata, Dockerfiles), not invocation patterns. The audit doesn't call SAW; it produces findings SAW can consume.

2. **Convention over coupling** - Shared file paths and naming conventions enable composition without dependencies. `Dockerfile.sandbox`, `docs/cold-start-audit.md`, `docs/IMPL/IMPL-*.md` are conventions, not requirements.

3. **Standalone utility** - Every skill works alone. dockerfile-sandbox-gen doesn't need cold-start-audit. The audit doesn't need SAW. Formula updater doesn't need release-engineer. Composition is optional, not mandatory.

4. **Graceful fallbacks** - Skills accept structured input OR manual alternatives. SAW reads audit findings OR requirements docs. Formula updater reads release metadata OR manual checksums. Each tool works even when the previous step didn't run.

5. **Strong data opinions, weak workflow assumptions** - We define what audit findings look like, not when you run audits. We define what IMPL docs contain, not how you trigger SAW. The artifacts are rigid; the workflows are flexible.

**Examples across skills:**

- **audit → SAW**: Findings report format (severity tiers, reproduction steps) maps directly to IMPL doc structure (wave priority, agent assignments)
- **release-engineer → homebrew-formula-updater**: Release metadata (version, SHA, checksums) feeds formula updates, but updater resolves checksums itself if needed
- **dockerfile-sandbox-gen → cold-start-audit**: Both implement the "audit-ready Dockerfile convention" independently
  - Generator produces: multi-stage build, non-root user with sudo, tool at `/usr/local/bin/<name>`
  - Audit requires: same contract, validates compliance, accepts any conforming Dockerfile
  - Contract documented in both repos, neither knows the other exists
  - Validation: `docker run --rm <image> bash -c "whoami && which <tool> && sudo -n true"`

This is why there's no "agentic-workflows orchestrator" tool. You compose by running skills in sequence and letting their outputs feed the next step. The orchestrator is you (or another AI agent reading these workflows).

---

## Infrastructure Patterns

These patterns enable deterministic skill composition at the technical level. Progressive disclosure without these patterns relies on agents reading references on demand, which is non-deterministic. Agents may skip critical documentation, read it after making mistakes, or fail to load it at all. These patterns make reference loading deterministic: content is present in context before execution starts.

### [Subcommand Dispatch](infrastructure/subcommand-dispatch.md)

**Repository:** [agentskills-subcommand-dispatch](https://github.com/blackwell-systems/agentskills-subcommand-dispatch)

Deterministic progressive disclosure at the orchestrator level via `triggers:` frontmatter field. When a user types `/saw scout`, the platform automatically loads scout-specific references before the orchestrator starts. Uses `UserPromptSubmit` hook with `additionalContext` injection.

**Example:** scout-and-wave loads scout-specific references when `/saw scout` is invoked, wave-specific references when `/saw wave` is invoked.

→ [Full documentation](infrastructure/subcommand-dispatch.md)

### [Agent References](infrastructure/agent-references.md)

**Repository:** [agentskills-agent-references](https://github.com/blackwell-systems/agentskills-agent-references)

Deterministic reference injection at the subagent level via `agent-references:` frontmatter field. When an orchestrator launches a wave-agent, the platform automatically injects wave-agent-specific references before the subagent starts. Uses `PreToolUse/Agent` hook with `updatedInput` modification. Supports conditional injection via `when:` regex patterns.

**Example:** scout-and-wave injects `wave-agent-completion.md` when launching wave agents, `scout-suitability-gate.md` when launching scouts. Production use reduced agent type prompt sizes by 40-60%.

→ [Full documentation](infrastructure/agent-references.md)

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

The three sandbox modes cover the full range of tool types: container for destructive operations, local for tools with redirectable state, worktree for directory-scoped tools. Output is a structured findings report (severity tier, affected area, reproduction steps) formatted to feed directly into SAW. Includes an Agent Skills-compatible `/cold-start-audit` skill - works with Claude Code, Cursor, GitHub Copilot, and other compatible tools.

### [dockerfile-sandbox-gen](https://github.com/blackwell-systems/dockerfile-sandbox-gen)

Dockerfile generator implementing the **audit-ready Dockerfile convention**: multi-stage builds with non-root users, sudo access, and standardized tool paths. Detects project language (Go, Rust, Python, Node.js) and generates container definitions that meet the explicit contract requirements for interactive testing environments.

Outputs to `Dockerfile.sandbox` with validation commands to verify contract compliance. Independently useful for any workflow requiring containerized tool builds. cold-start-audit consumes Dockerfiles meeting this contract via `--dockerfile` parameter, but neither tool depends on the other.

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

The audit's lack-of-context is the UX signal. An agent that can't figure out what to do next is faithfully simulating a new user hitting the same wall. SAW's pre-implementation check then filters already-fixed items before agents run, so each round only works on what's actually broken.

See [the full workflow doc](workflows/audit-fix-verify/) for the complete loop, sandbox mode selection, severity-to-wave mapping, and what the re-audit output means.

---

## Blog

Three-part series on scout-and-wave:

1. [A Coordination Pattern for Parallel AI Agents](https://blog.blackwell-systems.com/posts/scout-and-wave/): The pattern, the scout's role, and a worked example of wave structure and file ownership.
2. [What Dogfooding Taught Us](https://blog.blackwell-systems.com/posts/scout-and-wave-part2/): The audit-fix-audit loop in practice: overhead measurement, Quick mode for small scopes, and the bootstrap problem (using SAW to build SAW).
3. [Five Failures, Five Fixes](https://blog.blackwell-systems.com/posts/scout-and-wave-part3/): How the skill file decomposed from a monolith into composable pieces, and the scout prompt's bug tracker.
