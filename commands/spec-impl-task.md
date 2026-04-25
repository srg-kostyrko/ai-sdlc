---
description: Implement one vertical-slice task from tasks.md. Auto-ticks validation rows whose referenced tests pass; never auto-ticks the task checkbox.
argument-hint: <task-id> [<slug>]
---

You are implementing one vertical slice of an active change.

User input: $ARGUMENTS

## Step 1 — Parse and resolve

Parse:
- `task-id`: required. Format `<group>.<task>` (e.g. `1.2`, `2.1`). Reject other formats.
- `slug`: optional.

Resolve the slug:
- If a slug is provided, use `.sdlc/changes/<slug>/`. If that folder doesn't exist, list active changes and ask which one.
- If no slug is provided:
  - If exactly one folder exists under `.sdlc/changes/` (excluding `archive/`), use it.
  - Otherwise list the active changes and ask which one.

## Step 2 — Gate: tasks completeness

Refuse unless ALL hold. Print failures and stop on miss.

1. `.sdlc/changes/<slug>/tasks.md` exists.
2. `tasks.md` has at least one task.
3. Every task in `tasks.md` has `_Requirements:_` filled with ≥1 slug.
4. Every requirement slug in any `_Requirements:_` resolves (delta or living spec).
5. No `<!-- TODO -->` markers in `tasks.md`.
6. The requested `task-id` exists in `tasks.md`. Re-running on a task that is already ticked is **allowed** — the slice is re-implemented and the box stays ticked.
7. All tasks listed in this task's `_Depends:_` are ticked (`- [x]`).

## Step 3 — Read the task and its context

Read:
- The task block in `tasks.md`: title, `_Capabilities:_`, `_Requirements:_`, `_Boundary:_`, `_Depends:_`.
- The relevant delta files for slugs in `_Requirements:_`: `.sdlc/changes/<slug>/specs/<cap>/delta.md` for each cap. Extract the scenarios and criteria the task must make true.
- `.sdlc/changes/<slug>/design.md` `## Approach` and `## File Structure Plan`.
- Existing `.sdlc/changes/<slug>/validation.md` — to know which rows correspond to this task's requirements.
- Living `.sdlc/specs/<cap>/spec.md` for any MODIFIED slug (so you understand current behavior before modifying).

## Step 4 — Implement the slice

Plan briefly, then implement. Honor:

- `_Boundary:_` if listed — do not touch files outside the boundary unless absolutely necessary; surface deviations to the user.
- `_Capabilities:_` — implementation should affect only the listed bounded contexts.
- Vertical-slice discipline: at task end, every requirement in `_Requirements:_` should be demonstrably true. Setup work (migrations, scaffolding) folds into this slice — don't defer it.

Write tests as part of the implementation. For scenario rows, prefer integration or system-level tests; unit tests are appropriate for invariant criteria.

## Step 5 — Run tests and identify validation rows

After implementing:

1. Run the project's test suite, scoped if possible to the files you touched. Capture which tests passed and which failed.
2. For each row in `validation.md` whose `_Evidence:_` line references a test (`_Evidence:_ test://...`):
   - Check whether that test exists in the codebase and is green in this run.
   - If yes → tick the row (`- [x]`).
   - If no → leave unticked.
3. Manual rows (`_Evidence:_ manual — ...`) are **never** auto-ticked, even if related tests pass. They require human approval via `_Approved:_` line.
4. Rows whose `_Evidence:_` is empty are left unticked; the human will fill them in `/spec-validate` or directly.

## Step 6 — Auto-tick the task checkbox on success

If implementation completed without errors AND no test added or modified by this run is failing AND `_Boundary:_` (if listed) was not violated, **tick the task checkbox** (`- [ ]` → `- [x]`).

If any of those conditions failed (implementation raised an exception, a touched test is red, or boundary was violated), **leave the box unticked** and report what blocked the tick.

Validation rows are the contract for "requirements satisfied," not this checkbox. The checkbox is bookkeeping for "this slice has been implemented." If review of the diff reveals issues, the user can:
- Re-run `/spec-impl-task <id>` to re-implement (gate allows re-runs; the box stays ticked or re-ticks).
- Or ask Claude (in conversation) to untick the box and revise.

## Step 7 — Report

Print a structured summary:

```
Task <id>: <title>                                  [auto-ticked | left unticked: <reason>]
  Files changed:        <list>
  Tests added/updated:  <list>
  Tests run:            <count> passed, <count> failed
  Validation rows auto-ticked:
    - <req-slug> / <scenario or criterion>  (test://...)
  Validation rows still empty:
    - <req-slug> / <scenario or criterion>  (no test reference yet)

Next: review the diff. If issues, re-run /spec-impl-task <id> after telling
      me what to change, or ask me to untick + revise.
```

If any test failed or the boundary was violated, surface it prominently — the task box is left unticked and the report explains why.

## Constraints

- Never tick a manual validation row (`_Evidence:_ manual ...`) automatically. Manual rows always require `_Approved:_ <author> <date>`.
- Never modify deltas, design, proposal, or living specs from this command. The slice writes code; if you discover a real spec error mid-implementation, stop and ask the user to revise via `/spec-requirements` or `/spec-design`.
- Never run with `--no-verify`, skip tests, or bypass commit hooks.
