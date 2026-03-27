# agentic-workflows

[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)
[![Agent Skills](badge-agentskills.svg)](https://agentskills.io)

Composable agent skills extracted from real daily development patterns: cold-start audits, parallel fix execution, release automation, and sandbox generation.

> [!NOTE]
> All skills in this ecosystem follow the [Agent Skills](https://agentskills.io) open standard - compatible with Claude Code, Cursor, GitHub Copilot, Gemini CLI, and other Agent Skills-compatible tools. Install any skill in your preferred tool using the same directory structure.

These are the working patterns I rely on to:
- audit what new users hit first
- turn findings into safe, parallel fix waves
- ship releases and update downstream packages
- generate reproducible, isolated test environments

Each pattern produces structured output that feeds the next step in the loop.

## How this fits together

I use these patterns together in normal development work.

Audit findings become inputs to fix planning.
Fix execution produces changes to verify.
Release outputs become inputs to distribution updates.
Sandbox generation enables repeatable testing across all steps.

Each skill works independently, but they compose naturally because they solve adjacent problems in the same workflow.

---

## The development loop

**Audit → Plan → Execute → Verify → Release**

This replaces ad-hoc agent usage with a structured, repeatable development cycle.

---

## Common development patterns

### I need to see what a new user will hit first

Run `agentic-cold-start-audit` to produce a structured findings report with severity tiers and exact reproduction steps. Feed that report into `scout-and-wave` to plan and execute fixes in waves, then re-audit to verify what changed.

→ [Full workflow doc](workflows/audit-fix-verify/)

### I need to ship a release cleanly

Run `github-release-engineer` from the project repo. It handles pre-flight validation, changelog promotion, tagging, CI monitoring, artifact verification, and hands off release context to `homebrew-formula-updater`.

→ [Full workflow doc](workflows/release-publish/)

### I need a reproducible isolated environment

Run `dockerfile-sandbox-gen` to produce `Dockerfile.sandbox` for isolated, interactive testing environments that satisfy the audit-ready Dockerfile convention.

---

## Skills

### [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave) (SAW)

Parallel agent coordination protocol. A scout analyzes the codebase before any agent starts, producing file ownership maps and interface contracts. Agents then execute in waves with build and test gates between them.

Key guarantees: agents work concurrently without file conflicts; waves enforce dependency order; already-implemented work is filtered before agents launch; tests must pass before the next wave starts.

<details>
<summary>Execution flow diagram</summary>

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/diagrams/saw-scout-wave-dark.svg">
  <img src="docs/diagrams/saw-scout-wave-light.svg" alt="SAW scout + wave execution flow" style="max-width: 100%;">
</picture>

</details>

### [github-release-engineer](https://github.com/blackwell-systems/github-release-engineer)

Automated GitHub release pipeline. Runs pre-flight checks, detects or confirms the version, promotes the changelog entry, creates and pushes an annotated tag, watches CI, waits for release artifacts, and verifies the published release.

On completion, hands off release context (version, tag SHA, release URL, checksums) to companion skills such as `homebrew-formula-updater`.

### [homebrew-formula-updater](https://github.com/blackwell-systems/homebrew-formula-updater)

Updates a Homebrew tap formula to a new release. Resolves checksums from release assets, updates version, URL, and SHA256 fields, shows a diff, and commits and pushes the change.

Designed to be invoked by `github-release-engineer` at the end of a release, or run standalone.

### [agentic-cold-start-audit](https://github.com/blackwell-systems/agentic-cold-start-audit)

Cold-start UX auditing. Uses AI's lack of context as a feature by simulating a new user in a sandboxed environment and producing a structured findings report with severity tiers and exact reproduction steps.

Supports container, local, and worktree isolation modes.

### [dockerfile-sandbox-gen](https://github.com/blackwell-systems/dockerfile-sandbox-gen)

Generates `Dockerfile.sandbox` files that implement the audit-ready Dockerfile convention: multi-stage builds, non-root users, sudo access, and standardized tool paths.

Independently useful for any workflow requiring containerized tool builds.

---

## Workflows

### [Release-Publish](workflows/release-publish/)

1. Run `/release` from the project repo. The release engineer handles pre-flight, changelog promotion, tagging, CI monitoring, and artifact verification.
2. On success, the release engineer automatically invokes `/homebrew-formula-updater` with the release context. The formula updater reads checksums from the attached `checksums.txt`, updates the tap formula, and pushes.
3. Run `brew update && brew upgrade <tool>` to confirm the published formula installs cleanly.

The two skills compose because the release engineer's completion report carries exactly what the formula updater needs: version, tag SHA, release URL, and checksums, with no manual translation step.

See [the full workflow doc](workflows/release-publish/) for setup, configuration, and what to do when CI fails mid-release.

### [Audit-Fix-Verify](workflows/audit-fix-verify/)

1. Run `cold-start-audit` against your tool. Get a severity-tiered findings report with exact reproduction steps.
2. Feed the findings report to `SAW scout`. The scout reads audit findings alongside the codebase to produce file ownership, interface contracts, and a wave structure. Run `/saw wave --auto` to execute.
3. Re-run `cold-start-audit` against the fixed build. Compare against the original report: resolved, regressed, surviving, new.

The two tools compose because their artifact formats are compatible. The audit produces severity-tiered findings grouped by area; the SAW scout reads those tiers and groupings directly to decide wave order and agent assignments. No manual translation step.

The audit's lack-of-context is the UX signal. An agent that can't figure out what to do next is faithfully simulating a new user hitting the same wall. SAW's pre-implementation check then filters already-fixed items before agents run, so each round only works on what's actually broken.

See [the full workflow doc](workflows/audit-fix-verify/) for the complete loop, sandbox mode selection, severity-to-wave mapping, and what the re-audit output means.

---

## How composition works

These skills compose through compatible data formats, not hard dependencies.

Each tool does one thing well, produces structured output, and remains independently useful. The audit doesn't call SAW directly; it produces findings SAW can consume. The release engineer doesn't require the formula updater; it produces release context the updater can use. `dockerfile-sandbox-gen` and `agentic-cold-start-audit` both implement the same audit-ready Dockerfile convention without directly depending on each other.

**Key principles:**

- **Data contracts over API calls** - Tools agree on artifact formats (IMPL docs, findings reports, release metadata, Dockerfiles), not invocation patterns.
- **Convention over coupling** - Shared file paths and naming conventions enable composition without dependencies.
- **Standalone utility** - Every skill works alone. Composition is optional, not mandatory.
- **Graceful fallbacks** - Skills accept structured input OR manual alternatives.

There is no central orchestrator. Workflows are composed by running skills in sequence and letting their outputs feed the next step. The orchestrator is you (or another AI agent reading these workflows).

---

## Infrastructure Patterns

These patterns enable deterministic skill composition by ensuring the right references are present before execution begins, instead of relying on agents to discover them on demand.

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

