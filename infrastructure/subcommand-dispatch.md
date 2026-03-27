# Subcommand Dispatch

**Repository:** [agentskills-subcommand-dispatch](https://github.com/blackwell-systems/agentskills-subcommand-dispatch)

Deterministic progressive disclosure at the orchestrator level via `triggers:` frontmatter field.

## The Problem

Skills often need different context for different subcommands. When a user types `/saw scout`, the orchestrator needs scout-specific procedures. When they type `/saw wave`, it needs wave-specific procedures. Without deterministic loading:

- Agents may skip loading references entirely
- Agents may load them after making mistakes
- Orchestrator starts with generic context, loads specifics too late
- Skill behavior becomes non-deterministic

The alternatives are worse:
- **Load everything unconditionally** - bloats every invocation with unused context
- **Let agents read on demand** - fragile, non-deterministic

## The Solution

The `triggers:` frontmatter field declares which references to load for which subcommands. A `UserPromptSubmit` hook fires before the orchestrator starts, matches the user's input against trigger patterns, and injects matching references via `additionalContext`.

**Result:** References are present in the orchestrator's context before it takes its first step. No guessing, no "read this file" instructions that may be ignored.

## How It Works

### 1. Skill author declares triggers in SKILL.md

```yaml
---
name: saw
description: Parallel agent coordination
triggers:
  - pattern: "^/saw scout"
    inject: references/scout-procedure.md
  - pattern: "^/saw wave"
    inject: references/wave-procedure.md
---
```

### 2. Platform hook fires on user prompt submit

When the user types `/saw scout`, the `UserPromptSubmit` hook:
1. Reads all installed skills' `triggers:` declarations
2. Matches user input against each `pattern` regex
3. Collects matching `inject` file paths
4. Reads file contents
5. Injects via `additionalContext` parameter

### 3. Orchestrator receives enhanced context

The orchestrator's initial prompt includes:
- User's original message: `/saw scout "add feature"`
- Injected references: full content of `references/scout-procedure.md`
- Skill instructions: main `SKILL.md` body

All before the orchestrator takes its first step.

## Three-Layer Redundancy

The pattern works across platforms via three implementation layers:

### Layer 1: Hook (platform-native, deterministic)

Claude Code implementation:
```bash
~/.local/bin/inject_skill_context
```

Hook reads `triggers:` from all skills, matches user input, injects references.

### Layer 2: Script (vendor-neutral, orchestrator-initiated)

Skill includes `scripts/inject-skill-context`:
```bash
inject=$(bash ${SKILL_DIR}/scripts/inject-skill-context --input "$user_prompt")
# Platform passes enhanced prompt to orchestrator
```

SKILL.md instructs orchestrator: "Before you start, run this script and read the output."

### Layer 3: Routing table (convention-based, fallback)

SKILL.md explicitly lists subcommand mappings:
```markdown
When the user types `/saw scout`, read references/scout-procedure.md first.
When the user types `/saw wave`, read references/wave-procedure.md first.
```

Agent follows instructions manually.

## When to Use This

Use `triggers:` when:
- Your skill has multiple subcommands with distinct procedures
- Each subcommand needs 500-2000 lines of context
- Loading everything unconditionally would bloat the skill
- You need reliable reference loading before orchestrator starts

Don't use when:
- Single-command skill (no subcommand variants)
- References are short (< 500 lines total, just inline them)
- Context doesn't vary by invocation

## Example: scout-and-wave

scout-and-wave declares triggers for three subcommands:

```yaml
triggers:
  - pattern: "^/saw scout"
    inject: references/scout-procedure.md
    inject: references/scout-suitability-gate.md
  - pattern: "^/saw wave"
    inject: references/wave-procedure.md
  - pattern: "^/saw program plan"
    inject: references/program-planner-procedure.md
```

When `/saw scout "add feature"` is invoked:
- `scout-procedure.md` (17-step IMPL doc production) loads
- `scout-suitability-gate.md` (4-question assessment) loads
- Orchestrator starts with both procedures in context
- Scout doesn't need to "read the scout procedure" mid-execution

Without triggers, the orchestrator would either:
- Start with no context, load procedures too late
- Start with all procedures (scout + wave + program), waste context

## Installation

### For skill authors

1. Add `triggers:` to your SKILL.md frontmatter
2. Copy `scripts/inject-skill-context` from the pattern repo
3. Add fallback instruction to SKILL.md body:

```markdown
On platforms without UserPromptSubmit hook support, run:
bash ${SKILL_DIR}/scripts/inject-skill-context --input "<user-input>"
before starting and read the output.
```

### For platform implementers

Install the hook:
```bash
cp hooks/inject_skill_context ~/.local/bin/
chmod +x ~/.local/bin/inject_skill_context
```

Add to platform settings:
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {"type": "command", "command": "inject_skill_context"}
    ]
  }
}
```

Hook iterates all skill directories, delegates to each skill's script.

## Platform Support

- **Claude Code**: Full support (hook layer)
- **Cursor**: Compatible (hook layer via UserPromptSubmit equivalent)
- **VS Code / GitHub Copilot**: Compatible (script layer viable)
- **Gemini CLI**: Compatible (script layer viable)

Any Agent Skills-compatible platform can use the script layer or routing table fallback.

## Full Implementation

See [agentskills-subcommand-dispatch](https://github.com/blackwell-systems/agentskills-subcommand-dispatch) for:
- Complete hook implementation
- Portable bash script
- Spec proposal draft
- Integration examples
- Testing instructions
