---
description: Implement one or more vertical-slice tasks from tasks.md. Accepts task IDs (1.2) or whole groups (1). Auto-ticks green validation rows and the task checkbox; commits each successful slice.
argument-hint: <group | task-id>... [<slug>]
---

You are implementing one or more vertical slices of an active change.

User input: $ARGUMENTS

## Step 1 — Parse and resolve

Parse positional arguments. Each argument is one of:
- `<group>.<task>` — a single task ID (e.g. `1.2`)
- `<group>` — every task in that group, expanded in document order (e.g. `1`)
- A slug — any non-numeric token. At most one slug allowed.

Multiple targets may be passed (e.g. `/ai-sdlc:spec-impl-task 1.2 1.3 2.1`).

Resolve the slug:
- If a slug is provided, use `.sdlc/changes/<slug>/`. If that folder doesn't exist, list active changes and ask which one.
- If no slug is provided:
  - If exactly one folder exists under `.sdlc/changes/` (excluding `archive/`), use it.
  - Otherwise list the active changes and ask which one.

Resolve the targets:
- If any target is malformed (not matching `<int>` or `<int>.<int>`, or referencing a non-existent group/task), reject and stop. Do not list tasks.
- If no targets are given, list **unblocked tasks only** from `tasks.md` — those not yet ticked AND with every `_Depends:_` entry ticked. Show ID, title, `_Requirements:_`, grouped under their `## Group N — <name>` headings. If nothing is unblocked, say whether everything is done or blocked (with the blocking IDs) and stop. Otherwise ask which to implement and stop — do not proceed without explicit targets.
- Otherwise, expand any group-only target to its constituent task IDs in document order. The result is the **execution list**, in the order the user provided (groups expanded inline at their position).

## Step 2 — Gate: tasks completeness

Refuse unless ALL hold. Print failures and stop on miss.

**Global checks:**
1. `.sdlc/changes/<slug>/tasks.md` exists.
2. `tasks.md` has at least one task.
3. Every task in `tasks.md` has `_Requirements:_` filled with ≥1 slug.
4. Every requirement slug in any `_Requirements:_` resolves (delta or living spec).
5. No `<!-- TODO -->` markers in `tasks.md`.

**Per-task checks (apply to each task in the execution list):**
6. The task ID exists in `tasks.md`. Re-running on a task that is already ticked is **allowed** — the slice is re-implemented and the box stays ticked.
7. Every task in this task's `_Depends:_` is either already ticked (`- [x]`) OR appears earlier in the execution list.

If any per-task check fails, identify the offending task and stop without implementing anything — don't run a partial batch.

## Step 2.5 — Per-task loop

Steps 3 through 6 run **for each task in the execution list, in order**. After completing one task fully (read context → implement → run tests → record evidence → auto-tick), move to the next.

If any task fails (implementation raises an error, or a touched test is red), **stop the batch immediately**. The current task's checkbox stays unticked per Step 6; remaining tasks are reported as `skipped` in the final report.

A *requested slice merge* (Step 4) also halts the batch, since it needs a fresh execution list.

## Step 3 — Read the task and its context

Read:
- The task block in `tasks.md`: title, `_Capabilities:_`, `_Requirements:_`, `_Depends:_`.
- The relevant delta files for slugs in `_Requirements:_`: `.sdlc/changes/<slug>/specs/<cap>/delta.md` for each cap. Extract the scenarios and criteria the task must make true.
- `.sdlc/changes/<slug>/design.md` `## Approach` and `## File Structure Plan`.
- Existing `.sdlc/changes/<slug>/validation.md` — to know which rows correspond to this task's requirements.
- Living `.sdlc/specs/<cap>/spec.md` for any MODIFIED slug (so you understand current behavior before modifying).
- `.sdlc/changes/<slug>/findings.md` if present. Surface any open `FIX-BUG` or `FIX-MISMATCH` findings whose `req-slug` is in this task's `_Requirements:_` — these are the issues `/ai-sdlc:spec-review` flagged for this slice; address them as part of this re-implementation.

## Step 4 — Implement the slice

Plan briefly, then implement. Honor:

- `_Capabilities:_` — implementation should affect only the listed bounded contexts. If work spills into another bounded context, stop and ask the user before continuing.
- **Slice merging.** If implementation reveals that the right fix genuinely spans this slice and a later one in the batch — e.g. they share an abstraction that should land in one move — stop and ask the user whether to merge them. Don't silently absorb a downstream slice's scope. If the user agrees, proceed as one combined slice (single commit, both tasks tick).
- Vertical-slice discipline: at task end, every requirement in `_Requirements:_` should be demonstrably true. Setup work (migrations, scaffolding) folds into this slice — don't defer it.
- **Reuse over parallel structures.** If the natural fix is to extend an existing abstraction, do that. A "TODO: unify later" note next to ~10–20 LOC of parallel code is the wrong shape — extend the original.

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

If implementation completed without errors AND no test added or modified by this run is failing, **tick the task checkbox** (`- [ ]` → `- [x]`).

If either condition failed (implementation raised an exception, or a touched test is red), **leave the box unticked** and report what blocked the tick.

Validation rows are the contract for "requirements satisfied," not this checkbox. The checkbox is bookkeeping for "this slice has been implemented." If review of the diff reveals issues, the user can:
- Re-run `/ai-sdlc:spec-impl-task <id> <slug>` to re-implement (gate allows re-runs; the box stays ticked or re-ticks).
- Or ask Claude (in conversation) to untick the box and revise.

## Step 6.5 — Commit the slice

If Step 6 ticked the task AND there are uncommitted changes, create one commit. Skip if the box was left unticked or if the working tree has nothing new.

Stage only files this task touched: code, tests, `validation.md` updates, the `tasks.md` tick. Use `git add <path> <path> ...` with explicit paths — never `git add -A` or `git add .`. (If the user had WIP in a touched file, it folds into this commit; that is the user's responsibility to manage before invoking the command.)

Commit message — use a HEREDOC, subject = task title verbatim:

```
git commit -m "$(cat <<'EOF'
<task title>

Implements <slug> task <id>. Satisfies <req-slug>[, <req-slug>...].

<optional: 1–2 lines on approach if the diff alone isn't self-explanatory>
EOF
)"
```

Never use `--no-verify`, `--amend`, or skip pre-commit hooks. If a hook fails, address the underlying issue and create a new commit.

For a multi-task batch, each successful task commits as it completes — N successful tasks produce N commits.

## Step 7 — Report

Print one block per task in execution order, then a final batch line:

```
Task <id>: <title>  [✓ ticked | ✗ unticked: <reason> | — skipped]
  Validation: <ticked>/<in-scope> rows ticked (red: <slug/scenario>, empty: <count>)
  <if committed>: commit <sha>
  <if failures>: failing tests: <test://...>

Batch: <K>/<N> ticked, <S> skipped.
```

Omit `red:` and `empty:` parentheticals when both are zero. Omit `commit` and `failing tests:` lines when not applicable. The batch line always prints.

Diffs and added/changed test files are intentionally not enumerated — the user has `git diff`. The point of the report is task status, validation outcomes, and commit SHAs.

## Constraints

- Never tick a manual validation row (`_Evidence:_ manual ...`) automatically. Manual rows always require `_Approved:_ <author> <date>`.
- Never invent a test reference. Only fill `_Evidence:_ test://...` when the test actually exists in the codebase and demonstrates the scenario or criterion.
- Only modify validation rows whose requirement is in this task's `_Requirements:_`. Rows for other requirements belong to other slices.
- Never modify deltas, design, proposal, or living specs from this command. The slice writes code; if you discover a real spec error mid-implementation, stop and ask the user to revise via `/ai-sdlc:spec-requirements <slug>` or `/ai-sdlc:spec-design <slug>`.
- Never run with `--no-verify`, skip tests, or bypass commit hooks.
