---
description: Audit implementation against deltas + design for an active change. Records findings.md and a GO/NO-GO verdict. Required before /ai-sdlc:spec-archive.
argument-hint: [<slug>]
---

You are auditing the implementation of an active change against its deltas and design.

`/ai-sdlc:spec-review` is **read-only** on code, deltas, design, tasks, and validation. It writes only `findings.md` and reports. Re-running review is the resolution path — each run rewrites `findings.md` from scratch.

## Step 1 — Resolve the change

User input: $ARGUMENTS

- If `$ARGUMENTS` contains a slug, use `.sdlc/changes/<slug>/`. If that folder doesn't exist, list active changes and ask which one.
- If `$ARGUMENTS` is empty:
  - If exactly one folder exists under `.sdlc/changes/` (excluding `archive/`), use it.
  - Otherwise list the active changes and ask which one.

## Step 2 — Gate: implementation completeness

Refuse unless ALL hold. Print specific failures and stop on miss.

1. `.sdlc/changes/<slug>/tasks.md` exists, has ≥1 task, every task is ticked (`- [x]`).
2. `.sdlc/changes/<slug>/validation.md` exists.
3. Every row in `validation.md` is ticked (`- [x]`).
4. Every row has a non-empty `_Evidence:_` line.
5. Every row whose `_Evidence:_` starts with `manual` has an `_Approved:_` line.
6. No `<!-- TODO -->` markers in `proposal.md`, `design.md`, `tasks.md`, `validation.md`, or any `delta.md` for this change.

If a gate fails, print:

```
/ai-sdlc:spec-review refused. Implementation is not yet complete:
- tasks.md:14  task 3 is unticked
- validation.md:23  row missing _Evidence:_
Resolve via /ai-sdlc:spec-impl-task or /ai-sdlc:spec-validate.
```

## Step 3 — Read context

Read into main context:

- `.sdlc/changes/<slug>/proposal.md`
- All deltas under `.sdlc/changes/<slug>/specs/`
- `.sdlc/changes/<slug>/design.md`
- `.sdlc/changes/<slug>/tasks.md`
- `.sdlc/changes/<slug>/validation.md`
- Living capability specs touched by the deltas (`.sdlc/specs/<capability>/spec.md` and `GLOSSARY.md`)
- ADRs under `.sdlc/specs/<capability>/decisions/`, `.sdlc/decisions/`, and `.sdlc/changes/<slug>/decisions/`
- `.sdlc/steering/*.md` if present

Determine the diff scope:

- Compute the merge-base of current branch against the project's main branch (`main`, falling back to `master`). If neither exists, ask the user for the base ref.
- Diff range: `<merge-base>..HEAD` plus any uncommitted staged/unstaged changes.
- List changed files via `git diff --name-only <merge-base>` plus `git status --porcelain`. Exclude files under `.sdlc/` and `.claude/` — those are process artifacts, not implementation code.
- Capture both the merge-base SHA and the current HEAD SHA — both go into the verdict block.

Read every changed implementation file in full (current state, not just the diff) so review judges the code as it now stands.

## Step 4 — Code review pass

For each changed implementation file, assess against:

1. **Correctness** — logical errors, invalid assumptions, broken failure paths, state bugs.
2. **Scope discipline** — does the change stay within the modules named in `design.md ## File Structure Plan`, or touch unrelated code? Concrete file paths inside those modules are advisory; new files within a planned module are not a finding.
3. **Regression risk** — could existing behavior break? Edge cases handled?
4. **Design alignment** — for each public surface a requirement anchors, do public interface, return shapes, error handling, and naming match `design.md ## Approach` and the deltas? File location and naming choices that no requirement anchors are not a finding — impl is free to choose.
5. **Test adequacy** — tests present for changed behavior and the edge cases the deltas describe.

**Grounding rule:** before recording any finding, verify the code at the file:line you're about to cite. If you cannot find the code there, drop the finding.

## Step 5 — Acceptance mapping

For every ADDED and MODIFIED requirement, walk each `#### Scenario:` block and each `#### Criteria` bullet. For each, assess the **verification depth** the implementation reaches:

1. **Exists** — the relevant file/function/route is present.
2. **Substantive** — it contains real logic, not a stub or placeholder.
3. **Wired** — it is imported and called by the consuming module (not orphaned).
4. **Data-flowing** — at runtime it receives real data (not hardcoded empty values, not always-default branches).

Mark the deepest level reached. Anything below `Wired` is a finding (orphaned scaffolding, hollow implementation).

For every REMOVED requirement, confirm the prior behavior is genuinely gone — old code path deleted or guarded. Lingering dead code is a finding.

## Step 6 — Classify findings

Tag each finding with exactly one action type:

| Tag | Meaning | Heuristic |
|---|---|---|
| `FIX-MISMATCH` | Code doesn't match a design surface that **a requirement anchors** — fix the code | Return shape, public name, or wiring differs from delta/design AND a req-slug motivates that surface |
| `FIX-BUG` | Code produces wrong behavior — needs root-cause investigation | Null path, off-by-one, broken invariant, failing edge case |
| `DESIGN-GAP` | Design didn't cover a case the implementation hit — fix the design | Concurrency edge, error path, or data shape design didn't specify, that the impl had to invent |
| `SPEC-GAP` | A requirement is missing or ambiguous — needs spec clarification | Acceptance criterion silent on a real case the impl had to invent |

**Disambiguating FIX-MISMATCH vs no-finding for missing surface:** when code lacks a public surface that `design.md` declared, ask first: *which requirement slug anchors that surface?* If a slug in the deltas (or living spec) motivates it → `FIX-MISMATCH` (code must add it). If no slug motivates it → **not a finding** — the design surface was unanchored over-reach and the absent code is correct. The unanchored surface stays in `design.md` until the next `/ai-sdlc:spec-design` run cleans it up; it does not block GO.

If the same root cause produces multiple symptoms, record one finding citing the symptoms in its description.

## Step 7 — Verdict

The verdict is **GO** when ALL hold:

1. Zero open findings.
2. Every ADDED/MODIFIED scenario and criterion reaches `Wired` or `Data-flowing` depth.
3. Every REMOVED requirement's prior behavior is gone.

Otherwise **NO-GO**.

## Step 8 — Write findings.md

Rewrite `.sdlc/changes/<slug>/findings.md` from scratch. Do not read any previous file as input — re-running review is the resolution path.

Format:

```markdown
# <slug> — Review Findings

## Verdict
- Result: GO | NO-GO
- Reviewed: <YYYY-MM-DD HH:MM>
- Base ref: <merge-base SHA>
- Head ref: <current HEAD SHA>

## Findings

- [open] <TAG> — <req-slug or capability> / <file:line or —>
  <one-line description>
  <optional second line for additional context>
```

- Use `—` for `file:line` when the finding is conceptual (typical for `SPEC-GAP` and `DESIGN-GAP`).
- If verdict is GO, the `## Findings` heading is present but the list is empty.
- Findings are always recorded with `[open]` status. There is no `[resolved]` state — re-running review is what resolves a finding (it disappears from the next file).

## Step 9 — Report

Print a structured summary:

```
Review verdict: GO | NO-GO

Findings by tag:
  FIX-MISMATCH:  <count>
  FIX-BUG:       <count>
  DESIGN-GAP:    <count>
  SPEC-GAP:      <count>

Acceptance depth:
  Data-flowing:  <N>/<total>
  Wired:         <N>/<total>
  Substantive:   <N>/<total>
  Exists:        <N>/<total>
  Below Exists:  <N>/<total>   (missing artifacts)

Top findings:
  <tag> — <req-slug> / <file:line>
  <tag> — <req-slug> / <file:line>

Next:
  GO     → run /ai-sdlc:spec-archive when ready.
  NO-GO  → resolve and re-run /ai-sdlc:spec-review:
            FIX-BUG / FIX-MISMATCH → fix code, then /ai-sdlc:spec-impl-task <id> or direct edits.
            DESIGN-GAP             → revise via /ai-sdlc:spec-design.
            SPEC-GAP               → clarify via /ai-sdlc:spec-requirements.
```

## Constraints

- Read-only on code, deltas, design, tasks, validation. Only `findings.md` is written.
- Each run rewrites `findings.md`. Stale findings disappear when no longer flagged. There is no ignore mechanism — recourse is fix code, fix spec, or accept the NO-GO.
- Do not auto-fix findings. Surface them; the user decides which path to take.
- Grounding rule applies to every finding: cite real file:line; drop ungrounded findings.

## Error scenarios

- **Implementation not done (gate fail).** Print specific failures, suggest `/ai-sdlc:spec-impl-task` or `/ai-sdlc:spec-validate`.
- **No diff vs. main.** The branch has no implementation commits → review may still find issues in uncommitted changes; otherwise the verdict is trivially GO with zero findings (the change introduced no code, e.g. a spec-only refinement).
- **Main branch unknown.** If neither `main` nor `master` exists locally, ask the user for the base ref.
- **Finding spans multiple requirements.** Record once with the most-specific `req-slug`; mention the others in the description.
