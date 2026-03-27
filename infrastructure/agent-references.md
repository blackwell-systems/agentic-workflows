# Agent References

**Repository:** [agentskills-agent-references](https://github.com/blackwell-systems/agentskills-agent-references)

Deterministic subagent reference injection at agent launch time via `agent-references:` frontmatter field.

## The Problem

Orchestrator skills frequently launch typed subagents: a scout to analyze, a wave-agent to implement, a reviewer to verify. Each agent type needs type-specific procedures.

The tradeoff faced by skill authors:
- **Embed everything in agent type prompts** - bloated, 3000+ line prompts, mostly unused per invocation
- **Let agents read references on demand** - fragile, agents may skip them or read too late

The `triggers:` pattern solves this for orchestrators but can't reach subagents. `triggers:` fires at `UserPromptSubmit` (before orchestrator starts). Subagents launch mid-execution, long after that hook has fired.

## The Solution

The `agent-references:` frontmatter field declares which references to inject for which agent types. A `PreToolUse/Agent` hook fires before each subagent launches, matches the agent type, and injects matching references into the subagent's prompt via `updatedInput`.

**Result:** References are present in the subagent's context before it takes its first step. Agent type prompts stay slim (identity + role), heavy procedures stay in references, injection is deterministic.

## How It Works

### 1. Skill author declares agent-references in SKILL.md

```yaml
---
name: saw
description: Parallel agent coordination
agent-references:
  - agent-type: scout
    inject: references/scout-suitability-gate.md
  - agent-type: scout
    inject: references/scout-implementation-process.md
  - agent-type: wave-agent
    inject: references/wave-agent-completion-report.md
  - agent-type: wave-agent
    inject: references/wave-agent-worktree-isolation.md
  - agent-type: wave-agent
    inject: references/wave-agent-program-contracts.md
    when: "frozen_contracts_hash|frozen: true"
---
```

### 2. Platform hook fires before agent launch

When the orchestrator calls the Agent tool:
```python
Agent(
    subagent_type="wave-agent",
    prompt="Implement feature X in file Y..."
)
```

The `PreToolUse/Agent` hook:
1. Reads all installed skills' `agent-references:` declarations
2. Finds entries matching `subagent_type`
3. Tests any `when:` regex patterns against the agent prompt
4. Collects matching `inject` file paths
5. Reads file contents
6. Prepends to agent's prompt via `updatedInput`

### 3. Subagent receives enhanced prompt

The subagent's initial prompt includes:
- Injected references: `wave-agent-completion-report.md`, `wave-agent-worktree-isolation.md`
- Original prompt: "Implement feature X in file Y..."

All before the subagent takes its first step.

## Conditional Injection

The `when:` field enables conditional injection based on prompt content:

```yaml
- agent-type: wave-agent
  inject: references/wave-agent-program-contracts.md
  when: "frozen_contracts_hash|frozen: true"
```

Reference only injects when the orchestrator's prompt contains `frozen_contracts_hash` or `frozen: true`. Agents that don't need frozen contracts pay no context cost.

## Three-Layer Redundancy

The pattern works across platforms via three implementation layers:

### Layer 1: Hook (platform-native, deterministic)

Claude Code implementation:
```bash
~/.local/bin/inject_agent_references
```

Hook reads `agent-references:` from all skills, matches agent type, injects references.

**Supported platforms:**
- **Claude Code**: `PreToolUse/Agent` with `updatedInput`
- **Cursor**: `subagentStart` hook (testing)
- **VS Code**: Extension API (research needed)

### Layer 2: Script (vendor-neutral, orchestrator-initiated)

Skill includes `scripts/inject-agent-context`:
```bash
inject=$(bash ${SKILL_DIR}/scripts/inject-agent-context \
  --type wave-agent --prompt "$agent_prompt")
full_prompt="${inject}${agent_prompt}"
# Orchestrator passes full_prompt to Agent tool
```

SKILL.md instructs orchestrator: "Before launching each agent, run this script and prepend the output to the prompt."

### Layer 3: Routing table (convention-based, fallback)

SKILL.md explicitly lists agent type mappings:
```markdown
When launching a wave-agent, inject references/wave-agent-completion-report.md.
When launching a scout, inject references/scout-suitability-gate.md.
```

Orchestrator follows instructions manually.

## When to Use This

Use `agent-references:` when:
- Your skill launches multiple agent types with distinct procedures
- Each agent type needs 1000-3000 lines of type-specific context
- Loading everything into agent type prompts would bloat them
- You need reliable reference loading before subagent starts

Don't use when:
- Single-agent skill (no agent types)
- Agent type procedures are short (< 500 lines, just inline them)
- Context doesn't vary by agent type

## Example: scout-and-wave

scout-and-wave reduced agent type prompt sizes by 40-60% using this pattern:

**Before (monolithic):**
- scout prompt: ~2800 lines (identity + suitability gate + 17-step procedure + examples)
- wave-agent prompt: ~3200 lines (identity + completion format + isolation rules + build diagnosis + examples)

**After (with agent-references):**
- scout prompt: ~208 lines (identity + role + coordination protocol)
- scout references: 2 always-injected + 1 conditional
- wave-agent prompt: ~151 lines (identity + role + coordination protocol)
- wave-agent references: 3 always-injected + 1 conditional

### Actual declarations

```yaml
agent-references:
  - agent-type: scout
    inject: references/scout-suitability-gate.md
  - agent-type: scout
    inject: references/scout-implementation-process.md
  - agent-type: scout
    inject: references/scout-program-contracts.md
    when: "--program"
  - agent-type: wave-agent
    inject: references/wave-agent-worktree-isolation.md
  - agent-type: wave-agent
    inject: references/wave-agent-completion-report.md
  - agent-type: wave-agent
    inject: references/wave-agent-build-diagnosis.md
  - agent-type: wave-agent
    inject: references/wave-agent-program-contracts.md
    when: "frozen_contracts_hash|frozen: true"
```

## Installation

### For skill authors

1. Add `agent-references:` to your SKILL.md frontmatter
2. Copy `scripts/inject-agent-context` from the pattern repo
3. Add fallback instruction to SKILL.md body:

```markdown
On platforms without PreToolUse/Agent hook support, run:
bash ${SKILL_DIR}/scripts/inject-agent-context --type <agent-type> --prompt "<prompt>"
before launching each agent and prepend the output to the prompt.
```

### For platform implementers

Install the hook:
```bash
cp hooks/inject_agent_references ~/.local/bin/
chmod +x ~/.local/bin/inject_agent_references
```

Add to platform settings:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Agent",
        "hooks": [{"type": "command", "command": "inject_agent_references"}]
      }
    ]
  }
}
```

Hook iterates all skill directories, delegates to each skill's script.

## Platform Support

| Platform | Multi-Agent Support | Hook Support | Status |
|----------|---------------------|--------------|--------|
| **Claude Code** | Agent tool | `PreToolUse/Agent` | Production |
| **VS Code / Copilot** | `runSubagent` tool | Extension API | High potential |
| **Gemini CLI** | Custom subagents | Config files | Script layer viable |
| **Cursor** | Subagent hooks | `subagentStart`/`Stop` | Testing |

Platforms must support both Agent Skills and multi-agent/subagent capabilities for this pattern to apply.

## Vendor Compatibility Research

Full compatibility research across AI coding platforms: [docs/vendor-compatibility.md](https://github.com/blackwell-systems/agentskills-agent-references/blob/main/docs/vendor-compatibility.md)

Key findings:
- 4 platforms with both Agent Skills + multi-agent support
- Hook layer supported: Claude Code (production), Cursor (testing)
- Script layer viable: VS Code, Gemini CLI
- Platforms without Agent Skills: Cline, Aider, Continue, Windsurf (incompatible)

## Full Implementation

See [agentskills-agent-references](https://github.com/blackwell-systems/agentskills-agent-references) for:
- Complete hook implementation
- Portable bash script
- Spec proposal draft
- SAW production example (5 agent types, 14 reference entries)
- Vendor compatibility research
- Testing instructions
