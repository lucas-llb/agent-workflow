---
name: agent-workflow
description: Interactive agent workflow with router and specialist selection, following the flow workflow -> router -> analyst -> planner -> specialist (per task) -> final tasks (test-engineer, reviewer)
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
    description: (optional) force a specific specialist for all tasks, e.g. data-engineer, devops, frontend-react, frontend-angular, backend, mobile-flutter, security, infrastructure, unity, 3d-artist, game-developer, developer
---

> 🗿 **CAVEMAN MODE ACTIVE** — Use `/caveman` compressed communication in ALL responses to minimize token usage while preserving technical accuracy.

# Agent Development Workflow - Task-Driven Execution Mode

Task: $ARGUMENTS
Quality Level: ${quality:-pragmatic}

Flow: `workflow -> router -> analyst -> planner (enriches tasks) -> [per-task: specialist (TDD)] -> FINAL TASK A: test-engineer -> FINAL TASK B: reviewer (max 3 rounds)`

The **source of truth is always the task files** in `./docs/tasks/prd-[feature]/`. No `agent-plan-current.md` is created. The planner enriches the existing task files and each task is executed by its own specialist agent.

⛔ **`agent-reviewer` and `agent-test-engineer` are NOT part of the per-task loop.** They are not invoked between tasks. They exist only as the **two final tasks** created in PHASE 3, after all implementation tasks are done.

---

## Workflow Execution Logic

### Phase 0: Project Context (ALWAYS RUN FIRST)

Before any other step, verify and incorporate current project directives:

```
1. Look at project root for AGENTS.md -> if found, read and apply it.
2. If not found, look for CLAUDE.md -> if found, read and apply it.
3. Record project information: primary language, frameworks, conventions,
   constraints, code standards, folder structure, and specific instructions.
4. Pass this context to ALL subsequent agents (analyst, planner, specialist,
   and the final test-engineer / reviewer tasks).
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
   - The router may pick any implementation specialist, **including the game specialists**
     `unity`, `game-developer` and `3d-artist`.

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
       - The `specialist:` it assigns must be one of the implementation specialists in the
         PHASE 2 mapping — reviewer and test-engineer are **never** valid per-task specialists.
       - See `agent-planner` for enrichment format.

4. **PAUSE & PROMPT**: Display the enriched task list and STOP for explicit approval:
   ```
   Tasks enriched in ./docs/tasks/prd-[feature-slug]/
   Default Specialist: <name>
   Tasks ready for execution:
     [01] <task title> — specialist: <name>
     [02] <task title> — specialist: <name>
     ...
   After the last task, PHASE 3 will add:
     [NN]   Final test coverage — agent-test-engineer
     [NN+1] Final technical review — agent-reviewer (max 3 rounds)
   To proceed type: ok, continue, approve
   To cancel type: cancel
   ```

---

### PHASE 2: Execute Implementation Tasks (runs task by task)

When user types `ok`, `continue`, or runs the command again with enriched tasks present:

1. Read the feature slug and locate all `[num]_task.md` files in `./docs/tasks/prd-[feature-slug]/`, sorted by number.

2. Show start message:

   ```
   Starting task-by-task execution for <feature>...
   ```

3. **Task execution loop** — strictly sequential, one task at a time:

   Loop until all implementation task files are marked `status: done`:

   a. **Pick the first pending task** — the lowest-numbered `[num]_task.md` whose status is NOT `done`.
   - Never execute multiple tasks in parallel.
   - If all implementation tasks are `done`, proceed to **PHASE 3**.

   b. **Determine specialist for this task**:
   - Read the `specialist:` field from the task's `## Implementation Details` section.
   - If not set, use the default specialist confirmed in Phase 1.
   - Specialist mapping (implementation specialists — this is the full set the loop may dispatch):
     - `data-engineer` → `agent-data-engineer`
     - `devops` → `agent-devops`
     - `infrastructure` → `agent-infrastructure`
     - `security` → `agent-security`
     - `mobile-flutter` → `agent-mobile-flutter`
     - `frontend-react` → `agent-frontend-react`
     - `frontend-angular` → `agent-frontend-angular`
     - `backend` → `agent-backend`
     - `unity` → `agent-unity`
     - `3d-artist` → `agent-3d-artist`
     - `game-developer` → `agent-game-developer`
     - fallback `developer` → `agent-developer`

   > ⛔ `agent-reviewer` and `agent-test-engineer` are **not** in this mapping on purpose.
   > If a task file names one of them as `specialist:`, treat it as an enrichment error,
   > fall back to the default specialist, and fix the task file.

   > **Game specialists — first-class members of the loop; how to split the work:**
   > - `agent-unity` owns Unity-specific implementation: C# scripts, MonoBehaviours, ScriptableObjects, prefabs, URP/HDRP, Addressables, editor tooling, and Unity build settings.
   > - `agent-game-developer` owns engine-agnostic game architecture: ECS design, netcode/multiplayer, physics and AI systems, cross-platform abstraction, and performance budgets. Use it when the work spans engines or is not Unity-specific.
   > - `agent-3d-artist` owns art and technical-art deliverables: models, UVs, PBR materials, rigs, LODs, asset budgets, import settings, and DCC pipeline scripts.
   > - When a task mixes engine code and assets, split it: `agent-3d-artist` produces the asset spec, `agent-unity` (or `agent-game-developer`) integrates it.
   > - In a Unity project (`ProjectSettings/ProjectVersion.txt` + `Assets/` + `Packages/manifest.json`), `agent-unity` is the default specialist — never `agent-backend`, even though the code is C#.

   c. **SPECIALIST (implements via TDD — owns ALL tests for the task)**: Invoke the task's specialist agent for **only the current task**.
   - Provide the full task file content + PRD + TechSpec as context.
   - Specialist implements only that task following selected `quality`.
   - **TDD is mandatory** (`test-driven-development` skill): write failing tests first (red) → minimum implementation to pass (green) → refactor keeping green.
   - The specialist owns **every test level** for the task — unit (xUnit backend / RTL frontend), integration, and **E2E (Playwright)**. Test authoring during implementation is never delegated.
   - Map the task's PRD acceptance criteria to tests: every acceptance criterion in scope must have a covering test written by the specialist.
   - Per `CLAUDE.md`: E2E covers behavior only. Layout/design/color changes with no functional impact must NOT get E2E tests.
   - Run the suites for what changed — `dotnet test` (backend) and/or `npm run test:unit` + `npm run test:e2e` (frontend) — all must pass before the task can be marked done.
   - **Game tasks** (`unity`, `game-developer`, `3d-artist`): the test level is the Unity Test Framework (EditMode for logic, PlayMode for runtime behavior) or the engine's equivalent — not Playwright. Keep simulation logic decoupled from engine types so it stays unit-testable. Pure asset/visual deliverables from `agent-3d-artist` are validated against budgets and import settings via an optimization report instead of tests.
   - If a coverage gap cannot be closed (e.g., requires infra not available), document the reason in the task file — the final test-engineer task picks it up.

   d. **SELF-VERIFICATION (replaces the old per-task reviewer gate)**: the specialist applies `verification-before-completion` to its own work.
   - Build/compile clean, changed suites green, in-scope acceptance criteria covered.
   - ⛔ **Do NOT invoke `agent-reviewer` or `agent-test-engineer` here.** They run once, at the end, as the PHASE 3 tasks.

   e. **TASK STATUS UPDATE ⛔ BLOCKING GATE — execute before picking next task**:
   Once verification passes, immediately edit the task file `[num]_task.md`:
   1. Add or update a `status: done` field at the top of the file.
   2. Check [x] every subtask completed in the task
   3. Append a completion note: `> ✅ Implemented & verified — <one-line summary> — <timestamp>`
   4. Save the file.
   5. Notify the user: `✅ [num] <task title> complete. Moving to next...`
      ⛔ DO NOT pick the next task until `status: done` is confirmed in the task file.

   f. Move to the next pending task and repeat.

   > **No reviewer gate and no test-engineer gate inside this loop.** Implementation tasks flow
   > specialist → self-verification → done. Review and coverage audit happen once, at the end,
   > as the two PHASE 3 tasks.

4. When every implementation task is `status: done`, go to **PHASE 3**.

---

### PHASE 3: Final Quality Tasks (created after implementation ends)

When the last implementation task is marked `done`, **create two new task files** in `./docs/tasks/prd-[feature-slug]/`, numbered right after the last implementation task (e.g. if the last one is `07_task.md`, create `08_task.md` and `09_task.md`). They use the same task-file format as every other task (`status: pending`, `<requirements>` block, subtask checkboxes, reviewer criteria) so the run stays resumable after an interruption.

Execute them in order, sequentially.

#### Final Task A — `[NN]_task.md` "Final test coverage" → `agent-test-engineer`

- Read the PRD acceptance criteria plus every `[num]_task.md` test note, then map existing tests against them.
- Identify missing unit (xUnit backend / RTL frontend), integration and E2E (Playwright) coverage — for game projects, missing Unity Test Framework EditMode/PlayMode coverage instead.
- Write the missing tests following project conventions.
- Run the full affected suites — `dotnet test`, `npm run test:unit`, `npm run test:e2e`, or the Unity EditMode/PlayMode runs. All must pass.
- Gaps that genuinely cannot be closed are documented in the task file with the reason.
- Mark `status: done`, then continue to Final Task B.

> Test-engineer runs **before** the reviewer so the tests it authors are themselves covered by the final review.

#### Final Task B — `[NN+1]_task.md` "Final technical review" → `agent-reviewer`

⛔ **Hard limit: 3 review rounds. A 4th review pass is never allowed.**

```
Round 1 -> agent-reviewer reviews the whole feature (code + tests)
   approve, no blocking finding -> finalize now, skip remaining rounds
   needs changes                -> findings go to the owning specialist(s) -> fix -> Round 2

Round 2 -> agent-reviewer re-reviews
   approve                      -> finalize
   needs changes                -> findings go back to the specialist(s) -> fix -> Round 3

Round 3 -> agent-reviewer's FINAL review (no review happens after this one)
   whatever it reports          -> findings go back to the specialist(s) ONE last time -> fix
                                -> FINALIZE the task, with or without formal approval
```

Rules for this task:
- After each round, record the counter in the task file: `review_round: <N>/3`.
- Each fix round is executed by the specialist that owns the affected task (its `specialist:` field), never by the reviewer itself. The specialist re-runs the affected suites after fixing.
- After the round-3 fixes are applied, the task is finalized — **do not start a round 4** even if findings remain.
- Anything still open at that point is written into the task file under `## Post-review pending items` and reported to the user in the completion summary, never silently dropped.
- Mark `status: done` with the note `> ✅ Final review completed in <N>/3 rounds — <one-line summary> — <timestamp>`.

---

### Cleanup

- After both PHASE 3 tasks are `done`, notify the user with a completion summary that includes the review round count (`<N>/3`) and any `Post-review pending items`.
- No plan file to archive — task files are the permanent record.
- Show the completion message with a summary and links to all task files (implementation + the two final quality tasks).

---

## Notes on Agent Ordering & Responsibilities

- The **router** runs once in Phase 1 to establish the default specialist context, and may select any implementation specialist including `unity`, `game-developer` and `3d-artist`.
- The **analyst** feeds the **planner** with codebase constraints and spec gaps.
- The **planner** enriches task files — it does NOT create a plan file.
- The **specialist** is determined per-task from the task file's enrichment — different tasks may use different specialists. `agent-unity`, `agent-game-developer` and `agent-3d-artist` are first-class members of this set.
- The **specialist writes all tests via TDD**, including E2E (Playwright) or Unity EditMode/PlayMode.
- The **specialist self-verifies** (`verification-before-completion`) — there is no per-task quality gate by another agent.
- The **test-engineer** is no longer a per-task gate. It runs exactly once, as PHASE 3 Final Task A, auditing PRD coverage and writing whatever tests are still missing.
- The **reviewer** is no longer a per-task gate. It runs exactly once, as PHASE 3 Final Task B, capped at **3 review rounds**; after the third round's fixes the feature is finalized regardless of remaining findings, which are recorded as `Post-review pending items`.
- If execution is interrupted, resume by re-running the command: Phase 2 auto-detects enriched tasks and skips done ones, and Phase 3 resumes from whichever final task is still pending (reading `review_round:` to know which round to continue from).

---

## Edge cases and recommendations

- If router suggests a specialist but user cancels, abort cleanly and leave no partial state.
- For ambiguous tasks, router should suggest multiple candidates and allow user selection.
- If a task file has no `specialist:` field after enrichment, fall back to the default specialist.
- If a task file names `reviewer` or `test-engineer` as its `specialist:`, that is an enrichment error — fall back to the default specialist and correct the file.
- If execution is interrupted (e.g., token limit), rerun the command — it resumes from the first task without `status: done`, including the PHASE 3 tasks.
- If the PHASE 3 task files already exist, do not recreate them — resume them.
- If spec files do not exist and user rejects spec creation, abort the workflow cleanly.

---

## Skill Orchestration (AgentWorkflow)

Apply the following skills at the right points of the flow:

1. `test-driven-development` - mandatory for each specialist task before production edits, covering unit and E2E tests alike.
2. `dispatching-parallel-agents` - optional only for independent read-only investigations; keep implementation tasks strictly sequential.
3. `verification-before-completion` - mandatory before marking any task `status: done` (implementation tasks and both PHASE 3 tasks) and before final completion statements.
4. `finishing-a-development-branch` - mandatory after the two PHASE 3 tasks are done, to finalize merge/PR/cleanup choice.
