---
description: Implement one vertical-slice task from tasks.md. Records test evidence for in-scope validation rows and auto-ticks the green ones; never auto-ticks the task checkbox.
argument-hint: <task-id> [<slug>]
---

You are implementing one vertical slice of an active change.

User input: $ARGUMENTS

## Step 1 — Parse and resolve

Parse:
- `task-id`: required. A positive integer matching the task's position in `tasks.md` document order (e.g. `1`, `2`, `7`). Reject other formats.
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
- `.sdlc/changes/<slug>/findings.md` if present. Surface any open `FIX-BUG` or `FIX-MISMATCH` findings whose `req-slug` is in this task's `_Requirements:_` — these are the issues `/spec-review` flagged for this slice; address them as part of this re-implementation.

## Step 4 — Implement the slice

Plan briefly, then implement. Honor:

- `_Boundary:_` if listed — do not touch files outside the boundary unless absolutely necessary; surface deviations to the user.
- `_Capabilities:_` — implementation should affect only the listed bounded contexts.
- Vertical-slice discipline: at task end, every requirement in `_Requirements:_` should be demonstrably true. Setup work (migrations, scaffolding) folds into this slice — don't defer it.

Write tests as part of the implementation. For scenario rows, prefer integration or system-level tests; unit tests are appropriate for invariant criteria.

## Step 5 — Run tests, record evidence, tick green rows

Before any "tests passed" claim or auto-tick decision, follow the discipline in `${CLAUDE_PLUGIN_ROOT}/guidelines/verification-before-completion.md`: run the verification command fresh, read its full output (including exit code and failure count), and only then claim a result. No "should pass", no extrapolation from partial output.

After implementing:

1. Run the project's test suite, scoped if possible to the files you touched. Capture exit code, pass/fail counts, and the full failure list.

2. For each row in `validation.md` under a requirement listed in this task's `_Requirements:_`, decide what to do based on its current state:

   - **Manual row** (`_Evidence:_ manual — ...`) → never auto-touch. Manual rows always require human `_Approved:_`.
   - **Test-backed row, evidence empty** → if a test (one you just wrote, or one already in the codebase) demonstrates this scenario or criterion, fill `_Evidence:_ test://<relative-path>#<test name>` pointing at it. If that test is green in this run, tick the row. If red or absent, leave unticked.
   - **Test-backed row, evidence already populated** (`_Evidence:_ test://...`) → verify the test exists and is green; tick if so, leave unticked otherwise. Do not overwrite an existing pointer unless the referenced test has been removed or renamed.
   - **No applicable test** → leave `_Evidence:_` empty and unticked. Never invent a test reference.

3. Rows whose requirement is not in this task's `_Requirements:_` are out of scope for this slice — leave them alone, even if a touched test happens to cover them.

You are the one party who knows which test maps to which scenario at the moment of implementation; populate evidence as part of the slice rather than punting to a later human pass.

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
  Validation rows newly mapped:
    - <req-slug> / <scenario or criterion>  (test://...)   [ticked | red]
  Validation rows already mapped, re-verified:
    - <req-slug> / <scenario or criterion>  (test://...)   [ticked | red | stale]
  Validation rows still empty (in scope, no applicable test):
    - <req-slug> / <scenario or criterion>

Next: review the diff. If issues, re-run /spec-impl-task <id> after telling
      me what to change, or ask me to untick + revise.
```

If any test failed or the boundary was violated, surface it prominently — the task box is left unticked and the report explains why.

## Constraints

- Never tick a manual validation row (`_Evidence:_ manual ...`) automatically. Manual rows always require `_Approved:_ <author> <date>`.
- Never invent a test reference. Only fill `_Evidence:_ test://...` when the test actually exists in the codebase and demonstrates the scenario or criterion.
- Only modify validation rows whose requirement is in this task's `_Requirements:_`. Rows for other requirements belong to other slices.
- Never modify deltas, design, proposal, or living specs from this command. The slice writes code; if you discover a real spec error mid-implementation, stop and ask the user to revise via `/spec-requirements` or `/spec-design`.
- Never run with `--no-verify`, skip tests, or bypass commit hooks.
