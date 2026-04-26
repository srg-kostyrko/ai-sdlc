---
description: Refine proposal.md and capability deltas for an active change. Interviews the user, writes the initial draft to disk, then refines on-disk through self-check and grill-me.
argument-hint: [<slug>]
---

You are refining the requirements (proposal narrative + per-capability deltas) for an active change.

The flow is **interview → draft and write to disk → self-check → grill-me → glossary sweep → final re-check → report**. The initial draft writes to disk early so it's inspectable; later steps refine through on-disk edits. Structure (req- slugs, Scenario blocks, ADDED/MODIFIED/REMOVED partitioning) is your job, not the user's. Elicit understanding in plain language first, then formalize.

## Step 1 — Resolve the change

User input: $ARGUMENTS

- If `$ARGUMENTS` contains a slug, use `.sdlc/changes/<slug>/`. If that folder doesn't exist, list active changes and ask.
- If `$ARGUMENTS` is empty:
  - If exactly one folder exists under `.sdlc/changes/` (excluding `archive/`), use it.
  - Otherwise list the active changes and ask.

Pre-flight: the resolved folder must contain `proposal.md`. If not, suggest `/ai-sdlc:spec-propose`. The folder may have zero `specs/<capability>/delta.md` files right after `/ai-sdlc:spec-propose` — Step 4 seeds the first delta lazily.

## Step 2 — Read current state

Read into context:
- `.sdlc/changes/<slug>/proposal.md`
- Every `.sdlc/changes/<slug>/specs/*/delta.md` (may be zero — that's fine).
- For each capability already touched: `.sdlc/specs/<capability>/spec.md` and `.sdlc/specs/<capability>/GLOSSARY.md` (for slug resolution and term references). These may not yet exist for newly-introduced capabilities.
- `.sdlc/steering/*.md` if present (project-wide context).
- `.sdlc/changes/<slug>/findings.md` if present. Open `SPEC-GAP` findings drive the interview in Step 3 — turn each into a targeted question before asking the generic ones.

## Step 3 — Interview (always-on)

Ask 3–5 targeted open questions, **one at a time**, to fill gaps in `proposal.md` and surface the behaviors this change must guarantee. Skip a question only if the existing `proposal.md` already answers it concretely.

Cover, in roughly this order:
1. **Users and triggers.** Who interacts with this, and what causes the behavior to fire?
2. **Observable outcomes.** What must be true after the behavior runs that wasn't true before? Concrete, testable.
3. **Edges and failure modes.** What inputs, states, or timings break the happy path? What should the system do then?
4. **Invariants.** What must always hold regardless of trigger (e.g. data integrity, security, ordering)?
5. **Scope boundary.** What's explicitly *not* in this change?

Phrase questions in the user's domain language. Provide a recommended answer when you have grounded grounds for one (codebase evidence, prior `proposal.md` content, steering files). Wait for each answer before moving on. Do not ask the user to name `req-` slugs, capability names, or scenario shapes — those come from you in Step 4.

## Step 4 — Draft and write to disk

Author the initial draft, then write it to disk immediately. Subsequent steps (self-check, grill-me, glossary sweep) refine the files on disk through edits. The file stays inspectable at every checkpoint.

### proposal.md
- Fill `## Why`, `## What Changes`, `## Scope` from the interview answers, lightly edited for prose flow. Replace every `<!-- TODO -->` marker. Do not invent motivation the user did not give.

### Per-capability deltas
- Infer the most likely **bounded-context capability** from the interview content (e.g. `auth`, `billing`, `notifications`, `search`). If the change spans more than one capability, create one `delta.md` per capability, instantiated from `${CLAUDE_PLUGIN_ROOT}/templates/delta.md` with `{{CAPABILITY}}` substituted.
- For each behavior the user described, draft a requirement:
  - Title (short noun phrase) and `req-<slug>` (lowercase-dashed).
  - `**Why:**` — one line tying the requirement back to `proposal.md ## Why` or `## What Changes`.
  - **Triggered behavior** → `#### Scenario:` block with `WHEN <Domain Noun in Title Case> …` / `THEN observable outcome`. Use plain Title Case for domain terms; reach for opt-in `[[slug]]` markup only when prose would be ambiguous.
  - **Always-true rule** → `#### Criteria` bullet in EARS-Ubiquitous form (`The [System] shall [action]`). For full rules read `${CLAUDE_PLUGIN_ROOT}/guidelines/criterion-phrasing.md`.
- Place each in the right block:
  - `## ADDED Requirements` — slug not yet in the living spec.
  - `## MODIFIED Requirements` — slug already exists in the corresponding living `spec.md`.
  - `## REMOVED Requirements` — slug exists in living spec and this change drops it.
- For domain nouns surfacing in scenarios, draft `## ADDED Terms` entries (`### Term: Name {#term-slug}` + `**Definition:**` + `**Notes:**`). Pre-existing terms in living `GLOSSARY.md` need no entry — reference them by Title Case name.

### Write the initial draft

Once `proposal.md` and the per-capability deltas are drafted, write them to disk now:
- `.sdlc/changes/<slug>/proposal.md`
- Each `.sdlc/changes/<slug>/specs/<capability>/delta.md`. `mkdir -p` each `specs/<capability>/` parent before writing.

### Confirm capability inference

After writing, show the user a one-line summary per capability:

```
Capabilities inferred from your answers:
  auth          — 3 ADDED requirements, 2 ADDED terms
  notifications — 1 ADDED requirement
Confirm, or correct the capability split.
```

Apply corrections through edits to the on-disk files before proceeding.

## Step 5 — Self-check (silent, autonomous)

Run mechanical checks first, then judgment checks. Auto-repair within bounds; surface to the user only if a check fails after repair.

### Mechanical checks (must all pass)

1. `proposal.md` draft contains `## Why`, `## What Changes`, `## Scope`. None contains `<!-- TODO -->`.
2. The change has at least one ADDED requirement across its deltas.
3. Every requirement slug uses `{#req-slug}` and matches `^req-[a-z0-9][a-z0-9-]*$`.
4. Every term slug uses `{#term-slug}` and matches `^term-[a-z0-9][a-z0-9-]*$`.
5. No two ADDED requirement slugs collide across the change's deltas.
6. Every MODIFIED requirement slug exists in the corresponding living `spec.md`.
7. Every REMOVED requirement slug exists in the corresponding living `spec.md`.
8. Every opt-in `[[slug]]` reference (if any are used) resolves (living `GLOSSARY.md` OR the same change's `## ADDED Terms`).

### Judgment checks (apply after mechanical pass)

1. Each scenario is specific, observable, testable. Reject vague verbs ("supports X", "handles Y") without a measurable outcome.
2. Each `#### Criteria` bullet follows EARS-Ubiquitous form (see `${CLAUDE_PLUGIN_ROOT}/guidelines/criterion-phrasing.md`).
3. Each requirement's `**Why:**` ties back to `proposal.md ## Why` or `## What Changes`.
4. Per-capability deltas are coherent: every term in a scenario is pre-existing or has an `## ADDED Terms` entry; no orphans.
5. `proposal.md ## What Changes` lists at least one item that maps to each capability whose delta is non-empty.
6. Each scenario and criterion is verifiable from the subject's external boundary — observable behavior, externally visible state, or a published contract. Internal implementation choices (library, algorithm, internal schema field, internal method name) belong in `design.md`. Carve-out: when the change's purpose is to expose a contract to outside callers (new HTTP endpoint, wire format, CLI surface, SDK signature), naming the contract shape directly is correct.

### Repair loop

- If a check fails and the issue is local (typo, missing field, dangling reference, slug normalization, scope-mapping bullet missing): edit the file on disk and re-run.
- **Bounded to 2 repair passes.** After 2, stop and report the unresolved issue to the user — do not invent a fix.
- If the issue is a real ambiguity (contradictory user input, capability name unclear), stop on first occurrence and ask the user.

## Step 6 — Grill-me (mandatory)

Run the `grill-me` skill against the on-disk draft. This step **cannot** be skipped, regardless of change size.

Walk each branch of the decision tree:
- **Ambiguity** — scenarios where two reasonable readings give different outcomes.
- **Missing edges** — failure modes, race conditions, partial-state handling, retries, idempotency.
- **Scope tension** — items that look in-scope but the rationale points out-of-scope, or vice versa.
- **Invariant gaps** — implicit assumptions that should be explicit `#### Criteria` bullets.
- **Term collisions** — domain words used in two senses, terms that should be split or merged.
- **Tradeoff pressure** — where two requirements pull against each other, force a resolution.
- **Design leak** — items that name an internal implementation choice (library, algorithm, internal schema field, internal method name) where an observable property would describe the same requirement. Restate as observable, or defer to `design.md`. Skip this branch for items that name a contract this change exposes publicly.

Ask one question at a time, with your recommended answer grounded in codebase evidence or `proposal.md`. Label each question by its branch (e.g. `Design leak — Q1`, `Design leak — Q2`, then `Invariant gaps — Q1`). Question count is open — keep going within a branch until it resolves, then move to the next. Apply each agreed change as an on-disk edit as you go. Before concluding ask: "Anything else to challenge before we finalize?"

If grill-me produced changes, re-run Step 5's mechanical checks once more (no judgment-loop re-run — grill-me is the judgment loop).

## Step 7 — Glossary sweep

Invoke `spec-glossary-suggest` against each on-disk delta draft. For each surfaced candidate:
- **Capitalized phrase, likely term** — add an entry to `## ADDED Terms` (the prose reference is already correct).
- **Capitalized phrase, low-confidence** — leave alone unless the user flags it.
- **Unresolved opt-in `[[slug]]`** (rare) — typo or genuinely missing entry. Fix the markup or add to `## ADDED Terms`.

Apply changes as on-disk edits. Skip if zero candidates surface.

## Step 8 — Final mechanical re-check

After grill-me and glossary sweep, run Step 5's mechanical checks one final time against the on-disk files. If any fail, surface to the user.

## Step 9 — Report

By this point all checks have passed and the files are on disk. Report:

```
proposal.md:                   clean
specs/<cap>/delta.md:          clean (N requirements, M terms)
specs/<cap>/delta.md:          clean (N requirements, M terms)
Capabilities touched:          auth, notifications
Ready for /ai-sdlc:spec-design <slug>.
```

If Step 8 did not pass after the bounded repair passes, the draft remains on disk in its in-progress state. Report:

```
Refinement halted with unresolved issues:
  - <issue 1, with file:line>
  - <issue 2, with file:line>
The draft is on disk; edit directly or re-run /ai-sdlc:spec-requirements <slug>.
```

## Constraints

- Never write to `.sdlc/specs/` (living spec is read-only here).
- Never modify `tasks.md`, `design.md`, or `validation.md`.
- Slugs are immutable: REMOVE the old + ADD the new, never rename in place.
- One requirement per `### Requirement:` heading; avoid bundling unrelated behaviors.
- The user describes behaviors in plain language. You produce slugs, capability names, scenario shape, and EARS form.

## Error scenarios

- **Missing change folder.** `.sdlc/changes/<slug>/` does not exist → stop, suggest `/ai-sdlc:spec-propose`.
- **Multiple active changes (ambiguous).** `$ARGUMENTS` empty and >1 active change → list active slugs and ask.
- **Slug clash.** A drafted ADDED slug collides with an existing slug (living spec OR another delta in this change) → pick a different slug; if the user has a preferred name, ask.
- **MODIFIED/REMOVED of a non-existent slug.** Living spec doesn't have the targeted slug → reclassify as ADDED, or drop.
- **Capability inference rejected.** User corrects the capability split → re-bucket the on-disk deltas (move requirements between files; delete and recreate `specs/<capability>/` folders as needed) and re-run Step 5.
- **Grill-me reaches no shared understanding.** A challenge surfaces a contradiction the user cannot resolve → stop with the on-disk draft left in its current state; report the open question.
