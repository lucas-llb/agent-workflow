# Agent Workflow

Agent Workflow provides a task-driven, multi-agent workflow for turning a feature request into an approved implementation plan and executing it task by task.

The flow is:

```text
workflow → router → analyst → planner (enriches tasks)
  → [per-task: specialist (TDD) + self-verification]
  → FINAL TASK A: test-engineer (test coverage audit)
  → FINAL TASK B: reviewer (max 3 review rounds)
```

## Hosts

- **Claude Code** — run the command `/agent-workflow` from the project root.
- **Codex** — slash commands are not available; invoke the `agent-workflow` skill explicitly: `$agent-workflow implement feature X`.

## Installing the host files

The `Claude/` and `Codex/` folders contain the files that must be installed in each tool's user directory. Copy their contents into the matching hidden folder in your home directory; do not keep an extra `Claude/` or `Codex/` folder level.

For Claude Code, copy:

```text
Claude/agents/    → ~/.claude/agents/
Claude/commands/  → ~/.claude/commands/
Claude/skills/    → ~/.claude/skills/
Claude/templates/ → ~/.claude/templates/
```

For Codex, copy:

```text
Codex/agents/    → ~/.codex/agents/
Codex/skills/    → ~/.codex/skills/
Codex/templates/ → ~/.codex/templates/
```

On Windows, `~/.claude` means `C:\Users\<your-user>\.claude` and `~/.codex` means `C:\Users\<your-user>\.codex`. Merge the folders if they already exist so that other installed agents and skills are preserved.

After installation, open the project you want to work on and run the workflow from that project's root. The project's source code remains in the project repository; only the host files above belong in the user directories.

The workflow stores the source of truth in:

```text
docs/tasks/prd-[feature-slug]/
```

## Starting a new feature

Describe the feature when you first invoke the workflow:

```text
/agent-workflow "Add user authentication with email and password"
```

When no specification exists, the workflow creates the planning artifacts in order:

1. `prd.md` — product requirements and scope (created via the `create-prd` skill).
2. `techspec.md` — technical design and implementation approach (`create-techspec`).
3. `implementation_blueprint.md` — the planner's per-task implementation details, produced by the analyst + planner.
4. `tasks.md` and numbered `[num]_task.md` files — the implementation breakdown, already enriched with `## Implementation Details` sections and a `specialist:` per task (`create-tasks`).
5. Router confirmation of the default specialist.

The workflow pauses for user approval after the PRD, after the Tech Spec, and before task execution.

The resulting directory looks like this:

```text
docs/tasks/prd-user-authentication/
├── prd.md
├── techspec.md
├── implementation_blueprint.md
├── tasks.md
├── 01_task.md
└── 02_task.md
```

Bug fixes skip spec creation and are treated as a single task.

## Key observations

- **`prd.md` and `techspec.md` are the most important part for defining success** — they are what the whole implementation is measured against. Give them close attention.
- **Pay attention to the acceptance criteria** — always review them against the PRD before and during execution.
- **Model choice** — always use a more robust model for planning, task enrichment, and Final Tasks A and B. Use simpler models for coding.

## Continuing an existing feature

Re-invoke the workflow with the same feature description (or without any, if the current workflow is known by the host):

```text
/agent-workflow "Add user authentication with email and password"
```

The workflow detects the existing PRD and task files and resumes from the first task not marked `status: done`.

## Phases

### Phase 0 — Project context (always first)

Reads `AGENTS.md` (or `CLAUDE.md`), records the project's language, frameworks, conventions and constraints, and carries that context into every agent handoff. Project files take priority over workflow defaults.

### Phase 0.5 — Spec-driven context

Locates the feature spec in `docs/tasks/prd-[feature-slug]/`. Creates the PRD → Tech Spec → blueprint → tasks (as described above) for new features, or loads the existing documents.

### Phase 1 — Enrich tasks (runs once, before any implementation)

1. **Router** (`agent-router`) suggests the default specialist and asks the user to confirm, change, or cancel.
2. **Analyst** (`agent-analyst`) analyzes the codebase, PRD, Tech Spec and task files for gaps and constraints.
3. **Planner** (`agent-planner`) appends `## Implementation Details` to each task file, including its `specialist:`.
4. **Pause** — the enriched task list is shown and execution waits for explicit approval.

New features created via Phase 0.5 arrive already enriched, so Phase 1 skips straight to router + approval.

### Phase 2 — Execute implementation tasks

Tasks run strictly sequentially, one at a time, in numeric order:

- The specialist is read from each task's `specialist:` field (falling back to the router's default).
- The specialist implements the task via **mandatory TDD** (`test-driven-development`) and owns **every test level**: unit, integration, and E2E (Playwright) — or Unity EditMode/PlayMode for game tasks.
- The specialist runs the affected suites (`dotnet test`, `npm run test:unit`, `npm run test:e2e`) — all must pass.
- The specialist **self-verifies** (`verification-before-completion`). There is no per-task reviewer or test-engineer gate.
- The task file is marked `status: done` with a completion note before the next task is picked.

### Phase 3 — Final quality tasks

After the last implementation task, two new task files are created and executed in order:

- **Final Task A — "Final test coverage" → `agent-test-engineer`**: maps PRD acceptance criteria against existing tests, writes whatever is missing, and runs the full affected suites. It runs first so the tests it authors are covered by the final review.
- **Final Task B — "Final technical review" → `agent-reviewer`**: reviews the whole feature with a **hard limit of 3 review rounds**. Fixes between rounds are applied by the task's owning specialist, never by the reviewer. After round 3 the feature is finalized regardless of remaining findings — anything still open is recorded under `## Post-review pending items` and reported in the completion summary.

### Cleanup

A completion summary is shown (including the review round count `<N>/3` and any pending items), then `finishing-a-development-branch` is applied to finalize the merge/PR/cleanup choice.

## Specialists

Supported specialists (the full set Phase 2 may dispatch):

`data-engineer`, `devops`, `infrastructure`, `security`, `mobile-flutter`, `frontend-react`, `frontend-angular`, `backend`, `unity`, `3d-artist`, `game-developer`, and `developer` (fallback).

`agent-reviewer` and `agent-test-engineer` are never per-task specialists — they run only as the two Phase 3 final tasks.

## Optional parameters

```text
/agent-workflow "Add user authentication with email and password" --quality=strict --specialist=backend
```

- `--quality` — quality level (`pragmatic`, `balanced`, `strict`).
- `--specialist` — force a specific specialist for all tasks, skipping the router's suggestion.

## Interruptions and resume

If execution is interrupted, re-run the workflow. The task files are the permanent record:

- Phase 2 resumes from the first task without `status: done`; completed tasks are never repeated.
- If the Phase 3 task files already exist, they are resumed, not recreated — reading `review_round:` to know which review round to continue from.
