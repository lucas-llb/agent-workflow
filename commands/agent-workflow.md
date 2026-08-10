---
name: agent-workflow
description: Interactive agent workflow with router and specialist selection, following the flow workflow -> router -> analyst -> planner -> specialist -> reviewer
trigger: /agent-workflow
parameters:
  - name: task
    type: string
    required: false
    description: Feature to build or bug to fix (not needed when continuing)
  - name: quality
    type: string
    required: false
    description: Quality level (pragmatic, balanced, strict) - default is strict
  - name: specialist
    type: string
    required: false
    description: (optional) force a specific specialist for all tasks, e.g. data-engineer, devops, frontend-react, frontend-angular, backend, mobile-flutter, security, infrastructure, developer
---

> 🗿 **CAVEMAN MODE ACTIVE** — Use `/caveman` compressed communication in ALL responses to minimize token usage while preserving technical accuracy.

# Agent Development Workflow - Task-Driven Execution Mode

Task: $ARGUMENTS
Quality Level: ${quality:-pragmatic}

Flow: `workflow -> router -> analyst -> planner (enriches tasks) -> [per-task: specialist -> reviewer -> test-engineer]`

The **source of truth is always the task files** in `./docs/tasks/prd-[feature]/`. No `agent-plan-current.md` is created. The planner enriches the existing task files and each task is executed by its own specialist agent.

---

## Workflow Execution Logic

### Phase 0: Project Context (ALWAYS RUN FIRST)

Before any other step, verify and incorporate current project directives:

```
1. Look at project root for AGENTS.md -> if found, read and apply it.
2. If not found, look for CLAUDE.md -> if found, read and apply it.
3. Record project information: primary language, frameworks, conventions,
   constraints, code standards, folder structure, and specific instructions.
4. Pass this context to ALL subsequent agents (analyst, planner, specialist, reviewer).
5. If AGENTS.md or CLAUDE.md conflicts with workflow defaults,
   project-file instructions take priority.
```

> This phase guarantees a scope-aware workflow: behavior adapts to the current project, without assuming default stack or conventions.

---

### Phase 0.5: Spec-Driven Context (MANDATORY)

Before routing or analyzing, locate the spec documents for this feature:

1. **Identify the feature slug** from the task description (e.g., "user authentication" → `prd-user-authentication`).

2. **Check `./docs/tasks/prd-[feature-slug]/`** for:
   - `prd.md` → product requirements
   - `techspec.md` → technical specification
   - `tasks.md` + `[num]_task.md` files → task breakdown

3. **If NO spec exists (new feature):**
   - Invoke the `create-prd` skill → creates `./docs/tasks/prd-[feature]/prd.md`. Await user PRD approval before continuing.
   - Invoke the `create-techspec` skill → creates `./docs/tasks/prd-[feature]/techspec.md`. Await user TechSpec approval before continuing.
   - **ANALYST**: Invoke `agent-analyst` with PRD + TechSpec as context → analyze codebase, identify gaps and constraints.
   - **PLANNER (blueprint mode)**: Invoke `agent-planner` with analyst output + PRD + TechSpec → produces `./docs/tasks/prd-[feature]/implementation_blueprint.md` with per-task implementation details (files, tests, risks, specialist).
   - Invoke the `create-tasks` skill → reads `implementation_blueprint.md` as context → creates `./docs/tasks/prd-[feature]/tasks.md` and individual `[num]_task.md` files already enriched with `## Implementation Details` sections.
   - **ROUTER**: Invoke `agent-router` to confirm default specialist assignment.
   - PAUSE for user approval of enriched task list (same format as Phase 1 pause).
   - On approval → proceed directly to **PHASE 2** (skip Phase 1).

4. **If spec documents already exist** → load PRD, TechSpec, and all `[num]_task.md` files as context.

5. **Bug fixes** → skip spec creation; proceed directly to Phase 1 treating the bug description as the single task.

---

### Determining the Workflow Phase

Check whether the task files are already enriched:

```
Inspect ./docs/tasks/prd-[feature-slug]/01_task.md (or the first task file).
If it contains a "## Implementation Details" section -> tasks are enriched -> go to PHASE 2.
Otherwise -> tasks are not yet enriched -> go to PHASE 1.
```

---

### PHASE 1: Enrich Tasks (runs once, before any implementation)

> **Note:** For new features created via Phase 0.5, tasks arrive already enriched — Phase 1 skips analyst and planner and goes straight to ROUTER + PAUSE.

Execute enrichment phase sequentially:

```
Quality Level: ${quality:-pragmatic}
IMPORTANT: Execute agents SEQUENTIALLY - DO NOT run in parallel.
```

1. **ROUTER**: Use `agent-router` to analyze the overall task and suggest the default specialist.
   - If user provided `--specialist`, router respects it and skips suggestion.
   - Router prints suggested specialist and asks user to confirm:
     - Confirm (yes) -> proceed
     - Change specialist (provide name) -> router accepts and proceeds
     - Cancel -> abort workflow
   - The router's output is a **default specialist** used by the planner as context.
     Individual task files may specify a different specialist during enrichment.

2. **ANALYST + PLANNER (only if tasks are NOT yet enriched)**:
   - If `## Implementation Details` is already present in task files → **skip steps 2 and 3**.
   - Otherwise:
     - **ANALYST**: Use `agent-analyst` to analyze the task with awareness of the default specialist.
       - Load PRD, TechSpec, and all task files as context.
       - Identify gaps between spec assumptions and actual codebase state.
       - Output: concise analysis of current state + constraints passed to the planner.
     - **PLANNER**: Use `agent-planner` to **enrich each task file** (not to create a plan).
       - Planner reads every `[num]_task.md` and appends a `## Implementation Details` section to each.
       - It does NOT create `agent-plan-current.md`.
       - See `agent-planner` for enrichment format.

4. **PAUSE & PROMPT**: Display the enriched task list and STOP for explicit approval:
   ```
   Tasks enriched in ./docs/tasks/prd-[feature-slug]/
   Default Specialist: <name>
   Tasks ready for execution:
     [01] <task title> — specialist: <name>
     [02] <task title> — specialist: <name>
     ...
   To proceed type: ok, continue, approve
   To cancel type: cancel
   ```

---

### PHASE 2: Execute Tasks (runs task by task)

When user types `ok`, `continue`, or runs the command again with enriched tasks present:

1. Read the feature slug and locate all `[num]_task.md` files in `./docs/tasks/prd-[feature-slug]/`, sorted by number.

2. Show start message:

   ```
   Starting task-by-task execution for <feature>...
   ```

3. **Task execution loop** — strictly sequential, one task at a time:

   Loop until all task files are marked `status: done`:

   a. **Pick the first pending task** — the lowest-numbered `[num]_task.md` whose status is NOT `done`.
   - Never execute multiple tasks in parallel.
   - If all tasks are `done`, proceed to cleanup.

   b. **Determine specialist for this task**:
   - Read the `specialist:` field from the task's `## Implementation Details` section.
   - If not set, use the default specialist confirmed in Phase 1.
   - Specialist mapping:
     - `data-engineer` → `agent-data-engineer`
     - `devops` → `agent-devops`
     - `infrastructure` → `agent-infrastructure`
     - `security` → `agent-security`
     - `mobile-flutter` → `agent-mobile-flutter`
     - `frontend-react` → `agent-frontend-react`
     - `frontend-angular` → `agent-frontend-angular`
     - `backend` → `agent-backend`
     - fallback `developer` → `agent-developer`

   c. **SPECIALIST**: Invoke the task's specialist agent for **only the current task**.
   - Provide the full task file content + PRD + TechSpec as context.
   - Specialist implements only that task following selected `quality`.
   - Specialist must provide unit-test updates/results for the task.

   d. **REVIEWER (mandatory gate per task)**: Invoke `agent-reviewer` for the same task.
   - Reviewer validates code and the task-specific unit tests.
   - If reviewer returns `needs changes`, send feedback to specialist and retry the same task.
   - If reviewer returns `approve`, continue to test engineer.

   e. **TEST ENGINEER (mandatory coverage gate per task)**: Invoke `agent-test-engineer` for the same task.
   - Read the task's PRD acceptance criteria and map existing tests against them.
   - Identify missing unit tests (xUnit for backend, RTL for frontend) and E2E gaps (Playwright).
   - Write missing tests following project conventions.
   - Run `dotnet test` (backend changes) and/or `npm run test` + `npm run test:e2e` (frontend changes) — all must pass.
   - If coverage gaps remain unfixable (e.g., requires infra not available), document the reason.
   - If tests fail, send feedback to specialist and retry from step (c).
   - If all PRD scenarios are covered and tests pass, continue to status update.

   f. **TASK STATUS UPDATE ⛔ BLOCKING GATE — execute before picking next task**:
   After reviewer approves, immediately edit the task file `[num]_task.md`:
   1. Add or update a `status: done` field at the top of the file.
   2. Check [x] every subtask completed in the task
   3. Append a completion note: `> ✅ Reviewed & approved — <one-line summary> — <timestamp>`
   4. Save the file.
   5. Notify the user: `✅ [num] <task title> complete. Moving to next...`
      ⛔ DO NOT pick the next task until `status: done` is confirmed in the task file.

   f. Move to the next pending task and repeat.

4. **Cleanup**:
   - After all tasks are `done`, notify the user with a completion summary.
   - No plan file to archive — task files are the permanent record.

5. **Show completion message** with summary and links to the task files.

---

## Notes on Agent Ordering & Responsibilities

- The **router** runs once in Phase 1 to establish the default specialist context.
- The **analyst** feeds the **planner** with codebase constraints and spec gaps.
- The **planner** enriches task files — it does NOT create a plan file.
- The **specialist** is determined per-task from the task file's enrichment — different tasks may use different specialists.
- The **reviewer** is a mandatory per-task gate — validates code and unit tests before passing to test engineer.
- The **test engineer** is a mandatory per-task gate — audits PRD acceptance criteria coverage, writes missing tests, and runs all test suites before the task is marked done.
- If execution is interrupted, resume by re-running the command: Phase 2 auto-detects enriched tasks and skips done ones.

---

## Edge cases and recommendations

- If router suggests a specialist but user cancels, abort cleanly and leave no partial state.
- For ambiguous tasks, router should suggest multiple candidates and allow user selection.
- If a task file has no `specialist:` field after enrichment, fall back to the default specialist.
- If execution is interrupted (e.g., token limit), rerun the command — it will resume from the first task without `status: done`.
- If spec files do not exist and user rejects spec creation, abort the workflow cleanly.

---

## Skill Orchestration (AgentWorkflow)

Apply the following skills at the right points of the flow:

1. `test-driven-development` - mandatory for each specialist task before production edits.
2. `dispatching-parallel-agents` - optional only for independent read-only investigations; keep implementation tasks strictly sequential.
3. `verification-before-completion` - mandatory before marking a task `status: done` and before final completion statements.
4. `finishing-a-development-branch` - mandatory after all tasks are done to finalize merge/PR/cleanup choice.
