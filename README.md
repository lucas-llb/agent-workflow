# Agent Workflow

Agent Workflow provides a task-driven, multi-agent workflow for turning a feature request into an approved implementation plan and executing it task by task.

## Using `/agent-workflow`

Run the command from the project root. The workflow stores the source of truth in:

```text
docs/tasks/prd-[feature-slug]/
```

### 1. Create the PRD, Tech Spec, and tasks

For a new feature, describe the feature when you first invoke `/agent-workflow`:

```text
/agent-workflow "Add user authentication with email and password"
```

When no specification exists for the feature, the workflow creates the planning artifacts in order:

1. `prd.md` — product requirements and scope.
2. `techspec.md` — technical design and implementation approach.
3. `tasks.md` and numbered task files — the implementation breakdown.

The workflow pauses for approval after the PRD, after the Tech Spec, and before task execution. It also analyzes the codebase, enriches the task files with implementation details, and assigns a specialist to each task.

The resulting directory looks like this:

```text
docs/tasks/prd-user-authentication/
├── prd.md
├── techspec.md
├── tasks.md
├── 01_task.md
└── 02_task.md
```

Review and approve these documents before starting implementation.

### 2. Start or continue a PRD or task

After the planning files exist, invoke `/agent-workflow` again using the same feature description. Reusing the original description keeps the command pointed at the existing PRD folder:

```text
/agent-workflow "Add user authentication with email and password"
```

If the current workflow is already known by the host, you can continue it without repeating the task description:

```text
/agent-workflow
```

The workflow detects the existing PRD and task files, then:

- starts with the first task that is not marked `status: done`;
- executes tasks in numeric order, one at a time;
- delegates each task to its assigned specialist;
- runs the reviewer and test engineer gates;
- marks the task as done only after review and tests pass.

If execution is interrupted, run `/agent-workflow` again. It resumes from the first task without `status: done`; completed tasks are not repeated.

### Optional parameters

You can set the quality level or force a default specialist:

```text
/agent-workflow "Add user authentication with email and password" --quality=strict --specialist=backend
```

Supported specialist values include `backend`, `frontend-react`, `frontend-angular`, `data-engineer`, `devops`, `infrastructure`, `security`, `mobile-flutter`, and `developer`.

## Workflow at a glance

```text
New feature:
/agent-workflow
  → PRD
  → Tech Spec
  → task breakdown
  → approval
  → specialist
  → reviewer
  → test engineer
  → next task

Existing feature or interrupted run:
/agent-workflow
  → first pending task
  → specialist
  → reviewer
  → test engineer
  → resume or finish
```

The task files under `docs/tasks/prd-[feature-slug]/` are the permanent workflow record. No separate plan file is created.
