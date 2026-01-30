# Project Management: Infrastructure & Processes

**Category:** Architecture

---

This discussion documents the project management infrastructure and processes we've established for the TAU Platform ecosystem. The goal is alignment on how we plan, track, and coordinate work across repositories.

## GitHub Projects v2

We use a single GitHub Projects v2 board -- **"TAU Platform"** -- as the cross-repo coordination hub. All ecosystem repositories are linked to this board.

### Three-Tier Hierarchy

| Tier | GitHub Construct | Scope | Purpose |
|------|-----------------|-------|---------|
| **Project** | GitHub Projects v2 | Organization | Cross-repo board with all work items |
| **Phase** | SINGLE_SELECT custom field | Per project | Group items by development phase |
| **Item** | Project item (issue/PR/draft) | Per project | Individual work entry |

### Phase Field

Phases are implemented as a SINGLE_SELECT field named "Phase" on the project board. Current phases:

- **Backlog** -- Unscheduled work not yet assigned to a phase
- **Phase 1 - Foundation** -- Current active phase

Phases are added as the roadmap evolves. Each non-meta phase (not "Backlog" or "Done") gets a corresponding milestone on every linked repository.

### Milestone Convention

| Construct | Scope | Purpose |
|-----------|-------|---------|
| **Phase** | Cross-repo (project board) | Organize items across all repos |
| **Milestone** | Per-repo | Progress tracking within a single repo |

Rules:
- Milestone names match phase names exactly (e.g., "Phase 1 - Foundation")
- When assigning a phase to an issue, also assign the corresponding milestone
- "Backlog" phase does not get a milestone

## Label Convention

All TAU Platform repositories use a shared label taxonomy focused on work type categorization:

| Label | Description | Color |
|-------|-------------|-------|
| `bug` | Something isn't working correctly | `#d73a4a` |
| `feature` | New capability or functionality | `#0075ca` |
| `improvement` | Enhancement to existing functionality | `#a2eeef` |
| `refactor` | Code restructuring without behavior change | `#d4c5f9` |
| `documentation` | Documentation additions or updates | `#0e8a16` |
| `testing` | Test additions or improvements | `#fbca04` |
| `infrastructure` | CI/CD, build, tooling, project setup | `#e4e669` |

Labels are bootstrapped on new repos by cloning from `tau-platform`:

```bash
gh label clone tailored-agentic-units/tau-platform --repo tailored-agentic-units/<new-repo> --force
```

## Cross-Repo Coordination

### Linked Repositories

All ecosystem repos are linked to the TAU Platform project board. Currently:

- `tailored-agentic-units/tau-core`
- `tailored-agentic-units/tau-platform`

Additional repos will be linked as they are created (tau-orchestrate, tau-data, etc.).

### Issue Conventions

- **Titles**: Imperative verb phrase (e.g., "Add Audio protocol for speech-to-text transcription")
- **Labels**: At least one work-type label from the standard set
- **Phase assignment**: Set phase on the project board + milestone on the repo
- **Cross-references**: Link related issues across repos with URLs

### Discussions as Coordination

This repository (`tau-platform`) hosts GitHub Discussions for topics that span the ecosystem:

| Category | Purpose |
|----------|---------|
| **Announcements** | Updates from maintainers |
| **Architecture** | Design decisions, API design, cross-library concerns |
| **Ideas** | Feature proposals and new library suggestions |
| **Q&A** | Technical questions about TAU libraries |

Discussions follow the standards lifecycle: draft in `drafts/` --> publish as discussion --> formalize resolved decisions into `standards/`.

## Claude Code Automation

The `project-management` skill in tau-core automates common project operations via the `gh` CLI:

- Creating and configuring projects with phase fields
- Linking repositories to projects
- Adding issues to the board and assigning phases
- Resolving the ID chain (project ID --> field ID --> option ID --> item ID)
- Bootstrapping labels and milestones on new repos
- Viewing phase progress and cross-repo backlogs

The skill uses `gh project *` commands with `--format json` and `--jq` for structured output. As issue/PR lifecycle conventions are standardized (templates, title formats, label assignment, merge strategy), they will be incorporated directly into the `project-management` skill so that Claude can execute the full project management workflow from a single skill.

## Open Questions

1. **Issue templates**: Should we establish standardized issue templates for bugs, features, and improvements?
2. **PR conventions**: Should we formalize PR title format, required reviewers, or merge strategy (squash vs. merge)?
3. **Phase granularity**: How granular should phases be? Broad milestones (Foundation, Core Enhancements) or tighter iterations?
