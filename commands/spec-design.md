---
description: Refine design.md for an active change. Surfaces missing constraints, drafts in memory, runs self-check and grill-me, writes only when clean. Optionally dispatches a subagent to survey existing code.
argument-hint: [<slug>]
---

You are refining the design for an active change.

The flow is **resolve → gate → read context → constraints interview (skip-when-covered) → draft → self-check → grill-me → finalize**. Structure (section shape, ADR qualification, file-plan format) is your job. Elicit constraints in plain language when they're missing, then formalize.

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

If a gate fails, format the report as:

```
/ai-sdlc:spec-design refused. Reason(s):
- proposal.md:14  unresolved <!-- TODO --> in ## Scope
- specs/auth/delta.md:23  req- slug uses uppercase characters
Run /ai-sdlc:spec-requirements to resolve.
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
- `.sdlc/changes/<slug>/findings.md` if present. Open `DESIGN-GAP` findings drive the constraints interview in Step 3.5 — turn each into a targeted question before asking the generic ones. `DESIGN-GAP` has two flavors; ask the right question for each:
  - **Under-coverage** (design didn't specify a case the impl hit): ask which constraint, error path, concurrency edge, or data shape the design needs to add.
  - **Over-reach** (design named a surface no requirement anchors, code legitimately omitted it): ask drop-or-anchor — drop the surface from `## Approach` / `## File Structure Plan`, **or** stop and add the missing requirement via `/ai-sdlc:spec-requirements` first. Default recommendation is drop unless a real consumer for the surface exists today.

### Optional — dispatch a subagent for codebase survey

For brownfield work where the change touches existing code, dispatch a subagent (`subagent_type=Explore`) to survey the codebase. Skip for greenfield work or self-contained additions.

Example dispatch prompt:

> Survey existing code in <path-pattern> related to capability `<capability>`. Report under 200 lines: (1) current implementation patterns / interfaces this design should align with, (2) tech debt or constraints relevant to the proposed change, (3) integration points the change will need to honor. Quote file:line for the most load-bearing findings.

Wait for the findings summary before drafting. Use the summary in main context — do not load the raw exploration output.

## Step 3.5 — Constraints interview (skip-when-covered)

Before drafting, identify constraints the design must honor that aren't already captured in `proposal.md`, the deltas, or `.sdlc/steering/`. Ask only the questions whose answers are genuinely missing — skip silently when context already supplies them.

Candidate questions, asked one at a time, with your recommended answer based on codebase or steering evidence:

- **Budgets.** Are there latency, throughput, memory, or cost ceilings this design must fit under?
- **Tech preferences.** Anything you want to use here, or deliberately avoid? (Library choices, persistence strategy, sync vs async.)
- **Pattern alignment.** Existing patterns in the codebase to honor, or seams you want to deliberately break from?
- **Integration shape.** External systems or sibling capabilities this needs to talk to — sync calls, events, batch?
- **Reversibility appetite.** Are you willing to take a one-way decision here for speed, or does this need to be swappable later?

Stop as soon as remaining questions would be answered by content already in scope. If proposal/deltas/steering cover all five categories, skip the step entirely.

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
6. **Every public surface** named in `## Approach` or `## File Structure Plan` (exported types/interfaces, public methods, commands, routes, CLI flags, files-to-create) is **anchored by at least one requirement slug** in this change's deltas — or by an existing slug in a living spec touched by the deltas. Map each surface to its anchoring slug(s) explicitly during the check; a surface no slug motivates is design over-reach. Drop it from the draft, or stop and add the requirement via `/ai-sdlc:spec-requirements` before continuing.

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
- If the gate exposes a real **spec gap** (deltas can't support a coherent design), stop and ask the user to revise via `/ai-sdlc:spec-requirements` rather than papering over in the design.

## Step 5.5 — Grill-me (mandatory)

Run the `grill-me` skill against the in-memory design draft. This step **cannot** be skipped, regardless of change size.

Walk each branch of the decision tree. Focus on attacks that the mechanical gate cannot make:

- **Hidden assumptions in `## Approach`** — control flow that "obviously works" but only under conditions the design doesn't state.
- **Ownership drift in `## File Structure Plan`** — files placed in a capability the user wouldn't have chosen, or files that imply a refactor the proposal didn't authorize.
- **Unanchored surface** — for each public type, method, command, route, or file in `## Approach` and `## File Structure Plan`, name the requirement slug that motivates it. Surface that traces only to "future capability" or "extensibility" — not a slug — is pre-spec scaffolding. Drop it from the draft, or stop and add a requirement via `/ai-sdlc:spec-requirements` first. (The mechanical check #6 does the same audit; this prompt forces the conversation when the gate flagged something or when a surface feels speculative.)
- **Risks without mitigations** — every `## Risks` bullet should name a mitigation, an accepted budget, or a follow-up trigger; flag bullets that just describe pain.
- **ADR qualification** — decisions in `## Approach` that meet all three ADR criteria (hard to reverse + surprising + real trade-off) but weren't drafted as an ADR. Conversely, draft ADRs that don't meet all three should fold back into prose.
- **Tradeoff tensions** — pairs of design choices that pull against each other (latency vs durability, simplicity vs extensibility); force a resolution.
- **Goals/Non-Goals overlap** — anything that reads as both in and out of scope.

Ask one question at a time, with your recommended answer grounded in codebase evidence, deltas, or steering. Apply each agreed change to the in-memory draft as you go. Before concluding ask: "Anything else to challenge before we finalize?"

If grill-me produced changes, re-run Step 5's mechanical checks once more.

## Step 6 — Finalize

Once the review gate passes, write the in-memory draft to disk:

- `.sdlc/changes/<slug>/design.md`
- Any draft ADRs in `.sdlc/changes/<slug>/decisions/draft-*.md` (create the directory if needed).

Report:

```
design.md:                clean
ADRs drafted:             draft-totp-kms.md (1)
Subagent dispatched:      yes (codebase survey, <N> findings)   |   no (greenfield/simple)
Ready for /ai-sdlc:spec-tasks.
```

If the review gate did not pass after 2 repair passes, write **nothing** and report:

```
Design refinement halted with unresolved issues:
  - <issue 1, with file:line or section>
  - <issue 2>
Resolve and re-run /ai-sdlc:spec-design (or /ai-sdlc:spec-requirements if a spec gap was identified).
```

## Constraints

- Never write to `.sdlc/specs/` or `.sdlc/decisions/` (living state is read-only here).
- Do not invent design that the deltas don't motivate.
- Do not pre-assign ADR sequence numbers; drafts use slug-only filenames; archive assigns numbers.
- Keep `## Decisions` empty if no decision meets all three ADR criteria — most changes legitimately have zero ADRs.

## Error scenarios

- **Proposal incomplete (gate fail).** Print specific files/lines and stop. Suggest `/ai-sdlc:spec-requirements`.
- **Spec gap during design.** If the deltas cannot support a coherent design, stop and ask the user to clarify via `/ai-sdlc:spec-requirements`. Do not invent requirements in `design.md`.
- **ADR doesn't qualify.** Drop it; fold the rationale into `## Approach` prose.
- **Subagent returns vague findings.** If the survey is too generic to inform design (e.g. "the codebase has files"), ask the user to point at specific code paths instead of guessing.
- **Existing ADR conflict.** If the draft would contradict an accepted ADR, stop. Either the new design must change to honor the ADR, or a new ADR must supersede the old one (handle as a separate decision).
