---
description: Refine tasks.md for an active change. Drafts in memory, runs self-check and grill-me, writes only when clean. Vertical slices only.
argument-hint: [<slug>]
---

You are refining the task decomposition for an active change.

The flow is **resolve → gate → read context → draft → self-check → grill-me → finalize**. By tasks time the input (proposal, deltas, design) is firm — there is no pre-draft interview. Grill-me is the user's checkpoint before write.

## Step 1 — Resolve the change

User input: $ARGUMENTS

- If `$ARGUMENTS` contains a slug, use `.sdlc/changes/<slug>/`. If that folder doesn't exist, list active changes and ask which one.
- If `$ARGUMENTS` is empty:
  - If exactly one folder exists under `.sdlc/changes/` (excluding `archive/`), use it.
  - Otherwise list the active changes and ask which one.

## Step 2 — Gate: design completeness

Refuse unless ALL hold. Print specific failures and stop on miss.

1. `.sdlc/changes/<slug>/design.md` exists.
2. `design.md` contains required sections `## Approach`, `## Goals / Non-Goals`, `## File Structure Plan`, `## Risks`.
3. None of those sections contains `<!-- TODO -->` markers.
4. Every requirement slug mentioned in `design.md` resolves (against deltas or living specs).
5. Every draft ADR referenced in `design.md` exists at its declared path.
6. `## File Structure Plan` has concrete component/module entries (not placeholders).

If a gate fails, print the file:line and stop. Suggest `/ai-sdlc:spec-design`.

## Step 3 — Read context

Read into main context:

- `.sdlc/changes/<slug>/design.md` — especially `## Approach` (component flow) and `## File Structure Plan` (the files in scope)
- All deltas under `.sdlc/changes/<slug>/specs/` — collect every ADDED/MODIFIED/REMOVED requirement slug
- Existing `.sdlc/changes/<slug>/tasks.md` — used as merge base if present

## Step 4 — Draft tasks in memory

Decompose into vertical slices. Hold the draft **in memory** — do not write yet.

The hard discipline: **every task must satisfy at least one requirement slug**. No infrastructure, scaffolding, or setup-only tasks. Setup work folds into the first slice that needs it; the slice grows, and that's correct.

Decomposition rubric:

1. Identify the smallest set of slices that, taken together, satisfy all ADDED/MODIFIED/REMOVED requirements.
2. Cluster slices into logical groups (`## Group N — <name>`). A group collects tasks that share a capability, file area, or theme. Groups carry no execution semantics — `_Depends:_` alone encodes ordering. Name each group by the area of work ("user signup form"), not a phase ("Phase 1", "Setup").
3. Order groups, and tasks within each group, by suggested execution sequence (data-flow or risk-reduction order).
4. Assign each task an ID `<group>.<task>` matching its position (e.g. the second task of group 1 is `1.2`).
5. For each task:
   - Title: short imperative phrase describing the user-visible result.
   - `_Capabilities:_` capability slug(s) touched.
   - `_Requirements:_` `req-slug`(s) made true by this task.
   - `_Depends:_` task IDs this task genuinely depends on (within or across groups); `—` if none. Independence is conveyed by `—`.
6. Each task should be roughly day-sized. If smaller, consider merging; if larger, split into smaller end-to-end slices (not into a slice + a setup task).

Iterate with the user on the rubric output before finalizing.

## Step 5 — Review gate

Run mechanical checks first, then judgment checks.

### Mechanical checks (must all pass)

1. The draft has ≥1 group, and every group has ≥1 task.
2. Every group has a name (no `<!-- TODO -->` placeholder).
3. Every task ID is `<group>.<task>` matching its document position; IDs are unique.
4. Every task has `_Capabilities:_` with ≥1 capability slug.
5. Every task has `_Requirements:_` with ≥1 `req-slug`.
6. Every requirement slug in `_Requirements:_` resolves to an ADDED/MODIFIED slug in this change's deltas OR an existing slug in a living spec.
7. Every ADDED/MODIFIED requirement slug from the deltas appears in at least one task's `_Requirements:_` (no orphan requirements).
8. Every `_Depends:_` reference resolves to a task ID that appears earlier in the document (within or across groups).
9. No `<!-- TODO -->` markers remain.

### Judgment checks (apply after mechanical pass)

1. **Vertical-slice integrity.** Reject any task whose effect is only setup, scaffolding, migration, or "preparing X for later" — refold into the first downstream slice. Each task, when complete, should make at least one referenced requirement *demonstrably true*.
2. **Capability honesty.** `_Capabilities:_` matches the bounded contexts the slice actually touches. A slice that lists `auth, billing` but only modifies `auth` files is mislabeled.
3. **Day-sized target.** Flag any task that visibly exceeds 2–3 days or visibly takes <2 hours; ask whether to split or merge.
4. **Dependency honesty.** Every `_Depends:_` entry reflects a real ordering constraint (shared file, build artifact, runtime data). Spurious deps make the work look more sequential than it is; missing deps create silent breakage.
5. **Group coherence.** Each group's tasks share a capability, file area, or theme. Group names describe the area of work, not a phase or ordering. Tasks bundled together that don't belong, or tasks split across groups when they share scope, both warrant a re-cluster.

### Repair loop

- If a check fails and the issue is local to the draft, fix it and re-run the gate.
- **Bounded to 2 repair passes.** After 2, stop and report the unresolved issue.
- If the gate exposes a real **design gap** (e.g. a requirement that no slice can cleanly own), stop and ask the user to revise via `/ai-sdlc:spec-design` rather than fabricating a slice.

## Step 5.5 — Grill-me (mandatory)

Run the `grill-me` skill against the in-memory tasks draft. This step **cannot** be skipped, regardless of change size.

Walk each branch of the decision tree. Focus on attacks the mechanical gate cannot make:

- **Hidden setup work** — a slice that *looks* vertical but ships nothing user-visible until a later slice arrives. Refold into the first slice that delivers value.
- **Slice size honesty** — tasks that visibly exceed 2–3 days, or trivially small tasks that should fold into a neighbor. Propose where to cut or merge.
- **Capability drift** — `_Capabilities:_` lists a context the slice doesn't actually touch, or omits one it does.
- **Orphan or phantom requirements** — an ADDED/MODIFIED slug with no covering task, or a `_Requirements:_` reference that doesn't resolve. Surface as a real spec or design gap, not a slice fabrication.
- **Group coherence** — tasks bundled in one group that don't share capability/area/theme, or related tasks split across groups. Phase-style names ("Phase 1", "Backend setup") that describe ordering rather than the area of work.
- **Sequencing rationale** — task order that's arbitrary rather than driven by data flow or risk reduction. Spurious `_Depends:_` chains that pretend ordering exists where it doesn't.

Ask one question at a time, with your recommended answer grounded in design content, deltas, or codebase evidence. Apply each agreed change to the in-memory draft as you go. Before concluding ask: "Anything else to challenge before we finalize?"

If grill-me produced changes, re-run Step 5's mechanical checks once more.

## Step 6 — Finalize

Once the review gate passes, write the in-memory draft to `.sdlc/changes/<slug>/tasks.md`.

Report:

```
tasks.md:                    clean (<N> tasks across <M> groups)
Capabilities covered:        auth, notifications
Requirements covered:        <N>/<N> ADDED/MODIFIED  (no orphans)
Ready for /ai-sdlc:spec-validate <slug>.
```

If the review gate did not pass after 2 repair passes, write **nothing** and report:

```
Task refinement halted with unresolved issues:
  - <issue 1>
  - <issue 2>
Resolve and re-run /ai-sdlc:spec-tasks <slug> (or /ai-sdlc:spec-design <slug> if a design gap was identified).
```

## Constraints

- Reject any "infra-only", "scaffolding", or "setup" task. If proposed, refold into a story slice.
- Never modify `proposal.md`, `design.md`, deltas, or `validation.md`.
- Tasks are not commit boundaries automatically — implementation can produce multiple commits per task. The slice is the unit.
- Day-sized is a target, not a hard rule. 2 days is fine; 3+ deserves a split.

## Error scenarios

- **Design incomplete (gate fail).** Print specific files/lines, suggest `/ai-sdlc:spec-design`.
- **Orphan requirement.** A delta has an ADDED/MODIFIED requirement no task covers → ask the user: add a task that covers it, or move/remove it from the delta via `/ai-sdlc:spec-requirements`.
- **Phantom requirement.** A task references a `req-slug` that doesn't exist in any delta or living spec → ask the user: fix the typo, or add the requirement via `/ai-sdlc:spec-requirements`.
- **Setup-only task proposed.** User wants a task that doesn't satisfy any requirement → refuse; explain the vertical-slice rule; suggest folding into the next slice.
- **Task too big (3+ days).** Surface and ask whether to split into smaller end-to-end slices (never into slice + setup).
