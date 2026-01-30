# TAU Platform

Organizational coordination, planning, and standards for the [Tailored Agentic Units](https://github.com/tailored-agentic-units) ecosystem.

This repository serves as the central hub for:

- **Standards development** through GitHub Discussions
- **Planning artifacts** (diagrams, visual references)
- **Formalized standards** that govern cross-repo conventions

## Directory Conventions

| Directory | Purpose | Contents |
|-----------|---------|----------|
| `drafts/` | Discussion drafts | Markdown files staged for publication as GitHub Discussions |
| `standards/` | Formalized standards | Documents capturing resolved decisions from discussions |
| `planning/` | Visual artifacts | Diagrams, wireframes, and other visual planning materials |
| `archive/` | Historical artifacts | Pre-convention documents preserved for reference |

## Standards Lifecycle

```
Draft --> Discuss --> Formalize --> Operationalize
```

1. **Draft** -- Write discussion content in `drafts/` as markdown (`NN-kebab-case.md`)
2. **Discuss** -- Publish as a GitHub Discussion; team reviews and iterates
3. **Formalize** -- Extract resolved decisions into `standards/` documents (`kebab-case.md`)
4. **Operationalize** -- Create or update Claude Code skills to encode the standard

Each standard should have a corresponding Claude Code skill that enables AI agents to plan and implement in alignment with the standard.

## Discussion Categories

| Category | Purpose |
|----------|---------|
| **Announcements** | Updates from maintainers |
| **Architecture** | Design decisions, API design, cross-library concerns |
| **Ideas** | Feature proposals and new library suggestions |
| **Q&A** | Technical questions about TAU libraries |

## Naming Conventions

- **Drafts**: `NN-kebab-case.md` (e.g., `01-ontology.md`)
- **Standards**: `kebab-case.md` (e.g., `web-service-architecture.md`)
