# Claude Code Skills: AI-Assisted Development Standards

**Category:** Architecture

---

This discussion documents how we use Claude Code skills to encode organizational standards and accelerate development across the TAU ecosystem. The goal is alignment on how skills are structured, when they're created, how they mature from project-local to organization-wide, and which standards warrant skills in the first place.

## References

- [Claude Skills](https://code.claude.com/docs/en/skills)
- [Claude Plugins](https://code.claude.com/docs/en/plugins)

## What Are Claude Code Skills?

Claude Code skills are instruction files (SKILL.md) that provide domain-specific knowledge and patterns to an AI agent. When a skill's trigger conditions match the current task context, Claude automatically loads the skill and follows its instructions.

Skills live in a project's `.claude/skills/` directory and are configured in `.claude/settings.json`.

## Skills Established in tau-core

We currently have 6 skills in `tau-core/.claude/skills/`:

| Skill | Purpose |
|-------|---------|
| `go-patterns` | Go design patterns: interfaces, error handling, package structure, configuration lifecycle, modern idioms (Go 1.25+) |
| `github-cli` | GitHub CLI operations with 11 reference files covering issues, PRs, releases, search, workflows, API calls, gists, labels, discussions, secrets, variables |
| `project-management` | GitHub Projects v2: phase management, cross-repo backlogs, milestone conventions, label bootstrapping, composite workflows |
| `skill-creator` | Meta-skill for creating new skills: directory structure, frontmatter format, content patterns, settings integration |
| `tau-core-admin` | Contributing to tau-core: architecture, package hierarchy, provider/protocol extension patterns, testing strategy |
| `tau-core-dev` | Building applications with tau-core: installation, configuration, protocol usage, provider setup, mock testing |

## Skill Anatomy

### Directory Structure

```
.claude/skills/<skill-name>/
├── SKILL.md               # Required: main instructions (max 500 lines)
└── references/            # Optional: detailed reference docs
    └── detailed-guide.md
```

### Frontmatter

The YAML frontmatter controls when and how the skill is triggered:

```yaml
---
name: my-skill
description: >
  REQUIRED for <specific context>.
  Use when the user asks to "<action 1>", "<action 2>".
  Triggers: keyword1, keyword2, keyword3.

  When this skill is invoked, <explicit instructions>.
---
```

Key frontmatter patterns:
- **REQUIRED for** -- Declares the skill's scope
- **Use when** -- Lists trigger phrases in quotes
- **Triggers:** -- Comma-separated keywords for automatic detection
- **Inline instructions** -- What to do when invoked

### Content Structure

```markdown
# Skill Title

## When This Skill MUST Be Used

- Specific scenario 1
- Specific scenario 2

## Core Content

Organized by topic with code examples...

## Reference Tables

Quick-lookup tables for commands, patterns, etc.
```

## Project Settings

Skills are configured in `.claude/settings.json`:

```json
{
  "plansDirectory": "./.claude/plans",
  "permissions": {
    "allow": [
      "Bash(gh auth status*)",
      "Bash(gh issue list*)",
      "Bash(gh project list*)",
      "Skill(github-cli)",
      "Skill(go-patterns)",
      "Skill(project-management)",
      "Skill(skill-creator)",
      "Skill(tau-core-admin)",
      "Skill(tau-core-dev)"
    ]
  }
}
```

Key settings:
- **plansDirectory** -- Enables session continuity via plan files
- **permissions.allow** -- Pre-approves skills and read-only CLI operations
- Skills are listed in alphabetical order
- `Bash` permissions use glob patterns for read-only `gh` subcommands

## Session Continuity

Plan files in `.claude/plans/` enable work to be paused, resumed, and handed off across sessions or machines. When pausing work, request a context snapshot to be appended:

```markdown
## Context Snapshot - [YYYY-MM-DD HH:MM]

**Current State**: Brief description of where work stands
**Files Modified**: List of files changed this session
**Next Steps**: Immediate next action, subsequent actions
**Key Decisions**: Important decisions and rationale
**Blockers/Questions**: Unresolved issues
```

When resuming, the AI agent reads the plan file, reviews the latest snapshot, and continues from the documented next steps.

## Skill Maturity: Project to Organization

Skills follow a two-stage maturity path:

### Project-Local

A skill starts in a specific project's `.claude/skills/` directory, addressing that project's patterns. For example, `tau-core-admin` encodes how to extend tau-core's providers and protocols. Some skills are inherently project-specific and will remain at this level (e.g., `tau-core-admin`, `tau-core-dev`).

### Organizational Plugin

Once a project-local skill has proven useful and its patterns are stable, it should be promoted into an **organizational Claude Code plugin** -- a shared repository that can be installed into any project. This ensures:

- Consistent patterns across all ecosystem repositories
- Single source of truth for the skill (no drift between copies)
- New projects immediately benefit from accumulated organizational knowledge
- Updates propagate to all projects on plugin upgrade

Skills like `go-patterns`, `github-cli`, `project-management`, and `skill-creator` are strong candidates for promotion to the organizational plugin.

## When Standards Need Skills

Not every formalized standard requires a corresponding Claude Code skill. The determining factor is whether the standard directly affects how code is written, operations are performed, or decisions are made during development.

### Standards That Warrant Skills

Standards that encode **actionable patterns** -- things the AI agent must follow when writing code or executing operations:

| Standard | Why It Needs a Skill |
|----------|---------------------|
| Web service architecture | Guides service scaffolding, module structure, lifecycle patterns |
| Go design patterns | Directs interface design, error handling, package organization |
| Project management | Automates board operations, phase assignments, label bootstrapping |

These standards have concrete "do this, not that" guidance that an AI agent must follow during implementation.

### Standards That Inform Implicitly

Standards that establish **shared understanding** rather than actionable patterns. These permeate through other skills and documentation rather than requiring their own skill:

| Standard | Why It Doesn't Need a Skill |
|----------|----------------------------|
| Ontology (common vocabulary) | Terminology is used consistently across all other skills and documentation. The vocabulary is absorbed implicitly. |
| Iterative development mindset | Shapes how we approach planning and prioritization, but doesn't prescribe specific code patterns. |

These standards are valuable for team alignment but don't require explicit Claude Code enforcement. Their influence shows up in how other skills are written and how discussions are framed.

### The Guideline

Ask: **"Would an AI agent produce measurably different (better) output if it had this standard loaded as a skill?"**

- If yes -- the standard encodes patterns the agent should follow --> create a skill
- If no -- the standard informs human understanding and shapes other artifacts --> no skill needed

## The Compounding Effect

Skills encode institutional knowledge. Each new skill makes the AI agent progressively more capable within the ecosystem:

1. **Go patterns** -- The agent follows consistent code patterns
2. **Add project management** -- The agent can set up and manage cross-repo coordination
3. **Add web service architecture** -- The agent scaffolds services following proven patterns
4. **Add domain-specific skills** -- The agent understands each library's extension points
5. **Promote to organizational plugin** -- Every project in the ecosystem benefits immediately

Over time, the skill library becomes a comprehensive encoding of how the team works. New team members (human or AI) benefit from the accumulated knowledge immediately.

## Open Questions

1. **Plugin infrastructure**: When should we create the organizational plugin repository? After we have 3+ universally beneficial skills, or sooner?
2. **Plugin distribution**: Should the plugin be public (marketplace) or internal (`--plugin-dir` / git submodule)?
3. **Skill review process**: Should skill changes go through PR review like code, or are they lightweight enough for direct commits?
