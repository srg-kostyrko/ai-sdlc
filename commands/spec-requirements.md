---
description: Refine proposal.md and capability deltas for an active change. Drafts changes in memory, runs a review gate, writes only when clean.
argument-hint: [<slug>]
---

You are refining the requirements (proposal narrative + per-capability deltas) for an active change.

## Step 1 — Resolve the change

User input: $ARGUMENTS

- If `$ARGUMENTS` contains a slug, use `.sdlc/changes/<slug>/`. If that folder doesn't exist, list active changes and ask.
- If `$ARGUMENTS` is empty:
  - If exactly one folder exists under `.sdlc/changes/` (excluding `archive/`), use it.
  - Otherwise list the active changes and ask.

Pre-flight: the resolved folder must contain `proposal.md` and at least one `specs/<capability>/delta.md`. If not, suggest `/spec-propose`.

## Step 2 — Read current state

Read into context:
- `.sdlc/changes/<slug>/proposal.md`
- Every `.sdlc/changes/<slug>/specs/*/delta.md`
- For each capability touched: `.sdlc/specs/<capability>/spec.md` and `.sdlc/specs/<capability>/GLOSSARY.md` (for slug resolution on MODIFIED/REMOVED entries and term references). These may not yet exist for newly-introduced capabilities.

## Step 3 — Draft refinements in memory

Walk the user through the changes interactively. Hold proposed edits **in memory only** — do not write to disk yet.

### proposal.md
- For each `<!-- TODO -->` marker in `## Why`, `## What Changes`, `## Scope`, `## Rollout`: ask the user the targeted question; record the answer in the in-memory draft. Do not invent.
- Required sections must end TODO-free for the gate to pass.

### Each `delta.md`
- Audit `## ADDED Requirements`: each must have `### Requirement: TITLE {#req-slug}`, `**Why:**`, and either `#### Scenario:` (with `WHEN`/`THEN`) or `#### Criteria` (bullet invariants). Slugs are immutable once introduced.
- Audit `## MODIFIED Requirements`: every slug must already exist in the corresponding living `spec.md`. If not, propose moving it to ADDED.
- Audit `## REMOVED Requirements`: same — slug must exist in living spec.
- Term hygiene: every `[[term-slug]]` referenced must resolve to an existing living glossary entry OR an entry in this delta's `## ADDED Terms`. If unresolved, prompt the user: define here, fix the reference, or remove.
- Adding a NEW capability: hold a new `delta.md` (instantiated from `${CLAUDE_PLUGIN_ROOT}/templates/delta.md`) in memory; only write at finalization.

### Glossary deltas (`## ADDED Terms`, `## MODIFIED Terms`, `## REMOVED Terms`)
- Each new term needs `**Definition:**` (1–2 sentences) and `**Notes:**` (disambiguation/usage).

## Step 4 — Review gate

Run the mechanical checks first (no judgment), then the judgment checks. Repair locally and re-run on failures.

### Mechanical checks (must all pass)

1. `proposal.md` draft contains `## Why`, `## What Changes`, `## Scope`, `## Rollout`. None contains `<!-- TODO -->`.
2. Every requirement slug uses `{#req-slug}` and matches `^req-[a-z0-9][a-z0-9-]*$`.
3. Every term slug uses `{#term-slug}` and matches `^term-[a-z0-9][a-z0-9-]*$`.
4. No two ADDED requirement slugs collide across the change's deltas.
5. Every MODIFIED requirement slug exists in the corresponding living `spec.md`.
6. Every REMOVED requirement slug exists in the corresponding living `spec.md`.
7. Every `[[term-slug]]` reference resolves (living `GLOSSARY.md` OR the same change's `## ADDED Terms`).

### Judgment checks (apply after mechanical pass)

1. Each scenario is specific, observable, testable. Reject vague verbs ("supports X", "handles Y") without a measurable outcome.
2. Each `#### Criteria` bullet follows EARS-Ubiquitous form: `The [System] shall [action]` — explicit subject + `shall` + concrete action. Reject passive voice and implicit subjects. For full rules, examples, and antipatterns, read `${CLAUDE_PLUGIN_ROOT}/guidelines/criterion-phrasing.md` before this check.
3. Each requirement's `**Why:**` ties back to `proposal.md ## Why` or `## What Changes` — not a freestanding rationale.
4. Per-capability deltas are coherent: every term appearing in a scenario is either pre-existing or has a corresponding entry in `## ADDED Terms`; no orphan terms.
5. `proposal.md ## What Changes` lists at least one item that maps to each capability whose delta is non-empty (no silent capability touches).

### Repair loop

- If any check fails and the issue is local to the in-memory draft (typo, missing field, dangling reference): fix in the draft and re-run the review gate.
- **Bounded to 2 repair passes.** After 2, stop and report the unresolved issue to the user — do not invent a fix.
- If the issue is a real ambiguity (contradictory user input, capability name unclear), stop on the first occurrence and ask the user.

## Step 5 — Finalize

Once the review gate passes, write the in-memory draft to disk:

- `.sdlc/changes/<slug>/proposal.md`
- Each `.sdlc/changes/<slug>/specs/<capability>/delta.md` (including any newly-introduced capability deltas).

Report:

```
proposal.md:                   clean
specs/<cap>/delta.md:          clean (N requirements, M terms)
specs/<cap>/delta.md:          clean (N requirements, M terms)
Capabilities touched:          auth, notifications
Ready for /spec-design.
```

If the review gate did not pass after 2 repair passes, write **nothing** and report:

```
Refinement halted with unresolved issues:
  - <issue 1, with file:line>
  - <issue 2, with file:line>
Resolve and re-run /spec-requirements.
```

## Constraints

- Never write to `.sdlc/specs/` (living spec is read-only here).
- Never modify `tasks.md`, `design.md`, or `validation.md`.
- Slugs are immutable: REMOVE the old + ADD the new, never rename in place.
- One requirement per `### Requirement:` heading; avoid bundling unrelated behaviors.

## Error scenarios

- **Missing change folder.** `.sdlc/changes/<slug>/` does not exist → stop, suggest `/spec-propose "<one-line seed>"`.
- **Multiple active changes (ambiguous).** `$ARGUMENTS` empty and >1 active change → list active slugs and ask the user to specify.
- **No deltas yet.** `proposal.md` exists but no `specs/*/delta.md` → ask whether to create one (suggest the capability slug from the proposal narrative).
- **New capability requested.** User wants a delta for a capability not yet in the change → confirm the slug, then create the delta in memory.
- **Slug clash.** User proposes ADDING a slug that already exists (in living spec OR in this change's deltas) → reject; ask for a different slug.
- **Unresolved term reference.** A `[[term-slug]]` in scenario text doesn't resolve → prompt: define in `## ADDED Terms`, fix the reference, or remove.
- **MODIFIED/REMOVED of a non-existent slug.** Living spec doesn't have the targeted slug → propose moving the entry to ADDED, or drop it.
