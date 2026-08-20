---
name: agent-workflow
description: Runs the project's task-driven multi-agent workflow for features and bugfixes. Use when the user wants the old `/agent-workflow` behavior inside Codex.
---

# agent-workflow

Use this skill when the user asks to run the old `agent-workflow` command or wants the full project workflow with router, analyst, planner and specialists, closed by the two final quality tasks (test-engineer and reviewer).

## Invocation

Invoke this skill explicitly as `$agent-workflow`.

Examples:

- `$agent-workflow implement feature X`
- `$agent-workflow fix bug Y`
- `$agent-workflow continue the current feature workflow`

## Purpose

Execute the project workflow:

`workflow -> router -> analyst -> planner -> [per-task: specialist (TDD)] -> FINAL TASK A: test-engineer -> FINAL TASK B: reviewer (max 3 rounds)`

The source of truth is always the task files under `./docs/tasks/prd-[feature]/`.

## Operating Rules

1. Read project instructions first:
   - `AGENTS.md`
   - `CLAUDE.md` if needed
   - feature docs in `docs/`
2. Treat the user's prompt after `$agent-workflow` as the workflow task.
3. Use the custom agents explicitly:
   - `agent_router`
   - `agent_analyst`
   - `agent_planner`
   - task specialist from the task file (see Specialist Mapping)
   - `agent_test_engineer` and `agent_reviewer` **only as the two Phase 3 final tasks**
4. Apply the relevant skills during execution:
   - `test-driven-development`
   - `verification-before-completion`
   - `finishing-a-development-branch`
   - `dispatching-parallel-agents` only for independent read-only investigation
5. Keep implementation tasks sequential. Do not execute multiple pending task files in parallel.
6. ⛔ **`agent_reviewer` and `agent_test_engineer` are NOT per-task gates.** They are never invoked between implementation tasks. Each implementation task ends at specialist self-verification. Both agents run exactly once, as the two final task files created in Phase 3.
7. The task specialist writes every test itself via TDD — unit, integration, and E2E (Playwright), or Unity EditMode/PlayMode for game projects — before the production code. The final test-engineer task only audits and closes what is still missing.

## Workflow

### Phase 0: Project Context

Before any other step:

1. Read root project instructions.
2. Record project language, frameworks, constraints, conventions, and testing rules.
3. Carry that context into all agent handoffs.

### Phase 0.5: Spec-Driven Context

Locate the feature specification:

1. Infer the feature slug from the task.
2. Check `./docs/tasks/prd-[feature-slug]/` for:
   - `prd.md`
   - `techspec.md`
   - `tasks.md`
   - `[num]_task.md`
3. If no spec exists and this is a new feature:
   - use `create-prd`
   - use `create-techspec`
   - invoke `agent_analyst`
   - invoke `agent_planner`
   - use `create-tasks`
   - invoke `agent_router`
   - pause for approval before execution
4. If this is a bugfix, skip spec creation and treat the bug as a single task.

### Phase 1: Enrich Tasks

If task files are not yet enriched:

1. Invoke `agent_router` to choose the default specialist. It may pick any implementation specialist, including `unity`, `game-developer` and `3d-artist`.
2. Invoke `agent_analyst` with PRD, TechSpec, and task files.
3. Invoke `agent_planner` to enrich each `[num]_task.md` with implementation details. The `specialist:` it assigns must be an implementation specialist — `reviewer` and `test-engineer` are never valid per-task specialists.
4. Show the enriched task list and wait for explicit approval to continue.

### Phase 2: Execute Implementation Tasks

When approved:

1. Process task files in numeric order.
2. For each pending task:
   - determine the specialist from the task file
   - invoke the specialist agent for that task only, with TDD mandatory: failing tests first (unit + E2E when the task delivers a user-facing flow), then implementation
   - the specialist runs the affected suites (`dotnet test`, `npm run test:unit`, `npm run test:e2e`, or Unity EditMode/PlayMode) and they must pass
   - the specialist applies `verification-before-completion` on its own work — **no reviewer and no test-engineer are invoked here**
   - update the task file to `status: done` with the note `> ✅ Implemented & verified — <summary> — <timestamp>`
3. Repeat until all implementation tasks are `done`, then go to Phase 3.

### Phase 3: Final Quality Tasks

When the last implementation task is `done`, create two new task files in `./docs/tasks/prd-[feature-slug]/`, numbered right after the last implementation task (last is `07_task.md` -> create `08_task.md` and `09_task.md`). They use the standard task-file format (`status: pending`, `<requirements>`, subtask checkboxes) so the run stays resumable. Execute them in order.

#### Final Task A — `[NN]_task.md` "Final test coverage" -> `agent_test_engineer`

- Map PRD acceptance criteria and every task's test notes against the existing tests.
- Identify missing unit, integration and E2E coverage (or Unity EditMode/PlayMode coverage for game projects).
- Write the missing tests following project conventions.
- Run the full affected suites; all must pass.
- Document any gap that genuinely cannot be closed, with the reason.
- Mark `status: done` and continue to Final Task B.

> Test-engineer runs before the reviewer so the tests it authors are covered by the final review.

#### Final Task B — `[NN+1]_task.md` "Final technical review" -> `agent_reviewer`

⛔ **Hard limit: 3 review rounds. A 4th review pass is never allowed.**

```
Round 1 -> agent_reviewer reviews the whole feature (code + tests)
   approve, no blocking finding -> finalize now, skip remaining rounds
   needs changes                -> findings go to the owning specialist(s) -> fix -> Round 2

Round 2 -> agent_reviewer re-reviews
   approve                      -> finalize
   needs changes                -> findings go back to the specialist(s) -> fix -> Round 3

Round 3 -> agent_reviewer's FINAL review (no review happens after this one)
   whatever it reports          -> findings go back to the specialist(s) ONE last time -> fix
                                -> FINALIZE the task, with or without formal approval
```

Rules:
- After each round record `review_round: <N>/3` in the task file.
- Fixes are always applied by the specialist that owns the affected task, never by the reviewer. The specialist re-runs the affected suites after fixing.
- After the round-3 fixes, finalize — never start a round 4.
- Whatever is still open goes under `## Post-review pending items` in the task file and into the completion summary.
- Mark `status: done` with `> ✅ Final review completed in <N>/3 rounds — <summary> — <timestamp>`.

### Cleanup

After both Phase 3 tasks are done:

1. summarize completion, including the review round count and any `Post-review pending items`
2. apply `finishing-a-development-branch`

## Specialist Mapping

Implementation specialists — the full set Phase 2 may dispatch:

- `data-engineer` -> `agent_data_engineer`
- `devops` -> `agent_devops`
- `infrastructure` -> `agent_infrastructure`
- `security` -> `agent_security`
- `mobile-flutter` -> `agent_mobile_flutter`
- `frontend-react` -> `agent_frontend_react`
- `frontend-angular` -> `agent_frontend_angular`
- `backend` -> `agent_backend`
- `unity` -> `agent_unity`
- `3d-artist` -> `agent_3d_artist`
- `game-developer` -> `agent_game_developer`
- fallback -> `agent_developer`

⛔ `agent_reviewer` and `agent_test_engineer` are deliberately absent from this mapping — they are Phase 3 final tasks, not per-task specialists. A task file naming them as `specialist:` is an enrichment error: fall back to the default specialist and fix the file.

### Game specialists — how to split the work

- `agent_unity` owns Unity-specific implementation: C# scripts, MonoBehaviours, ScriptableObjects, prefabs, URP/HDRP, Addressables, editor tooling, and Unity build settings.
- `agent_game_developer` owns engine-agnostic game architecture: ECS design, netcode/multiplayer, physics and AI systems, cross-platform abstraction, and performance budgets. Use it when the work spans engines or is not Unity-specific.
- `agent_3d_artist` owns art and technical-art deliverables: models, UVs, PBR materials, rigs, LODs, asset budgets, import settings, and DCC pipeline scripts.
- When a task mixes engine code and assets, split it: `agent_3d_artist` produces the asset spec, `agent_unity` (or `agent_game_developer`) integrates it.
- In a Unity project (`ProjectSettings/ProjectVersion.txt` + `Assets/` + `Packages/manifest.json`), `agent_unity` is the default specialist — never `agent_backend`, even though the code is C#.
- Test level for game tasks is the Unity Test Framework (EditMode for logic, PlayMode for runtime behavior) or the engine's equivalent — not Playwright. Pure asset/visual deliverables from `agent_3d_artist` are validated against budgets and import settings via an optimization report instead of tests.

## Notes

- The old slash command `/agent-workflow` is not active in this Codex build.
- Use this explicit skill instead.
- If the user asks to continue a previous workflow, inspect existing task files and resume from the first task not marked `done`. If the Phase 3 task files already exist, resume them instead of recreating them, reading `review_round:` to know which review round to continue from.
