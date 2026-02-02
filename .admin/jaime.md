# Jaime's Notes

## Focus (2026-02-03)

1. Iron out `dev-workflow` skill and optimize integration with Project workflows + branch management. Refer to [github-kanban-board.html](../planning/github-kanban-board.html), as well as the [Branch Management](#branch-management) section below.
2. Work through `tau-core` Audio capability integration tasks
3. Flesh out the details for `tau-context` and `tau-tools` libraries.
  - `tau-context`: focuses on session + context management features for `tau-core`
  - `tau-tools`: focuses on using markdown-based skill definitions to execute tools, either natively available or custom-developed tools.
  - This will heavily influence the direction the rest of the platform is built out moving forward, so it will be imperative to take appropriate time planning and revising the API for these features.
  - May also need to consider the core processing loop (plan mode, todo list tooling, sub-agent execution, etc) to thoroughly test context and tool call optimization.
  - Tools will require security and permissions to ensure their safety and reliability of the system upon which the agent is operating.

## Branch Management

- `main`
  - `branch (hotfix)`: establishes [semantic versioning](https://semver.org) PATCH version increments for backward compatible bug fixes (MAJOR.MINOR.PATCH)
  - `feature branch`: branches named in the convention `MAJOR.MINOR_<phase_x>`, from which all task branches are modeled off of.
    - `task branch`: `<issue_number>-<title>`, encapsulates an individual issue.


Flow: `task branch` (pull)-> `feature branch` (pull)-> `main`

Hotfix Flow: `hotfix` (pull)-> `main` (merge)-> `feature branch`

## Joe's Workflow

- Planning Increments
- Quarter meetings for weekly planning
- Kanban w/ Sprints, coordinates Features to Tasks + Sub-Tasks
- **Requirement Batching**: Requirements (Epics) bundled into Feature Sets.
- **Dependency Identification**: break down into pseudo-tasks, then determine sequence of effort based on Task dependencies (Task X must be executed first because Task Y depends on it).
- **In-Depth Planning**: build out User Stories
- **IPM**: Point out user stories to determin complexity (high complexity, apart from greenfield topics, typically indicates a need to break down stories further).
- **Execution**: work through the current phase of development.
