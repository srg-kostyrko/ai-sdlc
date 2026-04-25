---
description: Refine design.md for an active change. Drafts in memory, runs a review gate, writes only when clean. Optionally dispatches a subagent to survey existing code.
argument-hint: [<slug>]
---

You are refining the design for an active change.

## Step 1 — Resolve the change

User input: $ARGUMENTS

- If `$ARGUMENTS` contains a slug, use `.sdlc/changes/<slug>/`. If that folder doesn't exist, list active changes and ask which one.
- If `$ARGUMENTS` is empty:
  - If exactly one folder exists under `.sdlc/changes/` (excluding `archive/`), use it.
  - Otherwise list the active changes and ask which one.

## Step 2 — Gate: proposal completeness

Refuse to proceed unless ALL hold. Print specific failures and stop on miss.

1. `.sdlc/changes/<slug>/proposal.md` exists.
2. `proposal.md` contains the required sections `## Why`, `## What Changes`, `## Scope`, `## Rollout`.
3. None of those sections contains `<!-- TODO -->` markers.
4. At least one `.sdlc/changes/<slug>/specs/*/delta.md` exists with a non-empty ADDED, MODIFIED, or REMOVED Requirements block.
5. Every requirement slug in the deltas matches `^req-[a-z0-9-]+$`.
6. Every `[[term-slug]]` resolves (living glossary OR same-change `## ADDED Terms`).

If a gate fails, format the report as:

```
/spec-design refused. Reason(s):
- proposal.md:14  unresolved <!-- TODO --> in ## Scope
- specs/auth/delta.md:23  term [[principal]] is not defined
Run /spec-requirements to resolve.
```

## Step 3 — Read context

Read into main context:

- `.sdlc/changes/<slug>/proposal.md`
- All deltas under `.sdlc/changes/<slug>/specs/`
- Existing `.sdlc/changes/<slug>/design.md` (if present — used as merge base)
- Living capability specs touched by the deltas (`.sdlc/specs/<capability>/spec.md` and `GLOSSARY.md`)
- Existing ADRs under `.sdlc/specs/<capability>/decisions/` and `.sdlc/decisions/` so the design doesn't contradict prior decisions
- **Project-wide steering** under `.sdlc/steering/`: read `product.md` (project goals + value), `structure.md` (module layout + conventions), `tech.md` (stack + constraints). Skip files that don't exist or contain only template placeholder text — those are unpopulated.
- **Brief-writing guideline:** read `${CLAUDE_PLUGIN_ROOT}/guidelines/durable-briefs.md` before drafting — its rules apply to `## Approach`, `## File Structure Plan`, and `## Risks`.

### Optional — dispatch a subagent for codebase survey

For brownfield work where the change touches existing code, dispatch a subagent (`subagent_type=Explore`) to survey the codebase. Skip for greenfield work or self-contained additions.

Example dispatch prompt:

> Survey existing code in <path-pattern> related to capability `<capability>`. Report under 200 lines: (1) current implementation patterns / interfaces this design should align with, (2) tech debt or constraints relevant to the proposed change, (3) integration points the change will need to honor. Quote file:line for the most load-bearing findings.

Wait for the findings summary before drafting. Use the summary in main context — do not load the raw exploration output.

## Step 4 — Draft design in memory

Iterate with the user. Author the design as an **in-memory draft** based on:

- `proposal.md` (especially `## What Changes`)
- The delta requirement slugs (the WHAT)
- Subagent findings, if dispatched
- Existing ADRs (do not contradict accepted decisions; if you must, the new decision needs an ADR that supersedes)

Required sections to author:

### `## Approach`
How the change is built. Components, control flow, data shapes. Reference requirement slugs. Explain the *how*; the deltas already explain the *what* — don't restate.

### `## Goals / Non-Goals`
**Goals** tie to `proposal.md ## What Changes`. **Non-Goals** explicitly exclude scope that could be misread (future work, integration points deferred, capabilities not touched).

### `## File Structure Plan`
Concrete files to create or modify, grouped by component. This is the seed of `_Boundary:_` values in `tasks.md` — vague file plans produce vague tasks. Use file paths, not abstractions.

### `## Risks`
What could go wrong. One bullet each: migration concerns, perf budgets, security implications, observability gaps, dependency risks. Include mitigation or accepted budget.

### `## Decisions` (optional)

Create an ADR **only** if the decision meets all three: hard to reverse, surprising without context, real trade-off.

If a decision qualifies:
1. Determine scope: per-capability (one capability) → draft path under `.sdlc/changes/<slug>/decisions/draft-<name>.md`, header `Affects: specs/<capability> — req-...`. System-wide (≥2 capabilities or no specific capability) → same draft path, header `Affects: system-wide`.
2. Hold the draft ADR in memory (instantiated from `${CLAUDE_PLUGIN_ROOT}/templates/adr.md` with `{{NUMBER}}` = `DRAFT` and `{{TITLE}}` filled). Fill `## Context`, `## Decision`, `## Consequences`. Optionally `## Alternatives Considered`.
3. Reference from `design.md`'s `## Decisions`: `- draft: decisions/draft-<name>.md — <one-line summary>`.

Optional sections (add only when relevant; not gate-required): Data Models, Testing Strategy, Components and Interfaces, Migration Strategy.

Do NOT write to disk yet.

## Step 5 — Review gate

Run mechanical checks first, then judgment checks. Repair locally and re-run on failures.

### Mechanical checks (must all pass)

1. `design.md` draft contains `## Approach`, `## Goals / Non-Goals`, `## File Structure Plan`, `## Risks`. None contains `<!-- TODO -->`.
2. Every requirement slug mentioned in the draft resolves to an ADDED/MODIFIED slug in this change's deltas OR an existing slug in a living spec.
3. Every draft ADR referenced in `## Decisions` has a corresponding draft path declared.
4. `## File Structure Plan` contains concrete file paths (no `TBD`, no empty placeholders).
5. Every component/file/module name mentioned in `## Approach` appears in `## File Structure Plan` (no orphan components).

### Judgment checks (apply after mechanical pass)

1. **Goals are specific** — each ties to a concrete item in `proposal.md ## What Changes`, not vague ambitions.
2. **Non-Goals are specific** — each excludes a misreading the reader could plausibly fall into, not generic "out of scope."
3. **Approach explains how, not what.** No restating of requirements in design.md.
4. **File Structure Plan implies ownership consistent with the change's capabilities.** Files outside the touched capabilities should be flagged as cross-cutting concerns or removed.
5. **Risks are concrete with mitigations or accepted budgets**, not platitudes ("might be slow").
6. **ADR qualification is honest.** If a draft ADR doesn't meet all three criteria (hard to reverse + surprising + real trade-off), demote it back to `## Approach` prose.

### Repair loop

- If a check fails and the issue is local to the draft, fix it and re-run the gate.
- **Bounded to 2 repair passes.** After 2, stop and report the unresolved issue.
- If the gate exposes a real **spec gap** (deltas can't support a coherent design), stop and ask the user to revise via `/spec-requirements` rather than papering over in the design.

## Step 6 — Finalize

Once the review gate passes, write the in-memory draft to disk:

- `.sdlc/changes/<slug>/design.md`
- Any draft ADRs in `.sdlc/changes/<slug>/decisions/draft-*.md` (create the directory if needed).

Report:

```
design.md:                clean
ADRs drafted:             draft-totp-kms.md (1)
Subagent dispatched:      yes (codebase survey, <N> findings)   |   no (greenfield/simple)
Ready for /spec-tasks.
```

If the review gate did not pass after 2 repair passes, write **nothing** and report:

```
Design refinement halted with unresolved issues:
  - <issue 1, with file:line or section>
  - <issue 2>
Resolve and re-run /spec-design (or /spec-requirements if a spec gap was identified).
```

## Constraints

- Never write to `.sdlc/specs/` or `.sdlc/decisions/` (living state is read-only here).
- Do not invent design that the deltas don't motivate.
- Do not pre-assign ADR sequence numbers; drafts use slug-only filenames; archive assigns numbers.
- Keep `## Decisions` empty if no decision meets all three ADR criteria — most changes legitimately have zero ADRs.

## Error scenarios

- **Proposal incomplete (gate fail).** Print specific files/lines and stop. Suggest `/spec-requirements`.
- **Spec gap during design.** If the deltas cannot support a coherent design, stop and ask the user to clarify via `/spec-requirements`. Do not invent requirements in `design.md`.
- **ADR doesn't qualify.** Drop it; fold the rationale into `## Approach` prose.
- **Subagent returns vague findings.** If the survey is too generic to inform design (e.g. "the codebase has files"), ask the user to point at specific code paths instead of guessing.
- **Existing ADR conflict.** If the draft would contradict an accepted ADR, stop. Either the new design must change to honor the ADR, or a new ADR must supersede the old one (handle as a separate decision).
