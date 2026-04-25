---
description: Start a new change proposal. Creates .sdlc/changes/<slug>/ with first-draft proposal, design skeleton, tasks skeleton, and initial delta against one capability.
argument-hint: <slug> "<one-line description>"
---

You are creating a new ai-sdlc change proposal.

User input: $ARGUMENTS

## Step 1 — Parse and validate arguments

Extract:
- `slug`: the first whitespace-separated token. Must match `^[a-z0-9][a-z0-9-]*$` (lowercase alphanumeric + dashes; must start alphanumeric). Reject empty, slashes, spaces, uppercase.
- `description`: everything after the slug (strip surrounding quotes). Must be non-empty.

If invalid or missing, print a one-line usage hint and stop:
`/spec-propose <slug> "<one-line description>"`

## Step 2 — Pre-flight

- Verify `.sdlc/` exists at the working directory. If not: print `Run /sdlc-init first.` and stop.
- Verify `.sdlc/changes/<slug>/` does NOT exist. If it does: print `Change <slug> already exists.` and stop.
- Check `.sdlc/changes/archive/` for any folder ending with `-<slug>`. If a match exists, warn the user (slug clashes with archived history) and ask whether to proceed.

## Step 3 — Pick the initial capability

Look at the description. Identify the most likely **bounded-context capability** affected (e.g. `auth`, `billing`, `notifications`, `search`).

- If the description clearly implies one capability, use it.
- If unclear or the description spans multiple, **ask the user** which capability to seed the initial delta against. They can add more capabilities during `/spec-requirements`.
- The capability slug must be lowercase-dashed.

Do not create any directories under `.sdlc/specs/` — capabilities are seeded lazily, only by `/spec-archive`.

## Step 4 — Create the change folder

```
.sdlc/changes/<slug>/
.sdlc/changes/<slug>/specs/<capability>/
```

(Do not create `decisions/` yet — created lazily when an ADR is drafted.)

## Step 5 — Author first-draft artifacts

Read each template from `${CLAUDE_PLUGIN_ROOT}/templates/` (this plugin's `templates/` directory; the path can be determined from the location of this command file — its parent's parent is the plugin root). Substitute `{{CHANGE_SLUG}}` with the slug and `{{CAPABILITY}}` with the chosen capability slug.

### `.sdlc/changes/<slug>/proposal.md`

Adapt `templates/proposal.md`:
- **Why**: 2–3 sentences expanding the user's one-line description into a motivation. Be specific about the problem; do not invent constraints not implied by the input. If you cannot fill it confidently, leave the template `<!-- TODO -->` marker and note that `/spec-requirements` will refine.
- **What Changes**: 1–3 numbered items naming the concrete changes. Stay close to the description — do not invent additional changes. If the description implies only one change, write one item. Leave `<!-- TODO -->` for any item you cannot ground in the input.
- **Scope**: best-effort In/Out lists based on the description. Leave `<!-- TODO -->` for items you cannot infer.
- **Rollout**: default to `Reversible until /spec-archive merges deltas. Ship user-visible changes behind a feature flag.` unless the description implies a specific rollout shape.

### `.sdlc/changes/<slug>/design.md`

Write the template **as-is**, only substituting `{{CHANGE_SLUG}}`. Do not invent design content; design is `/spec-design`'s job after the proposal stabilizes.

### `.sdlc/changes/<slug>/tasks.md`

Write the template **as-is**, only substituting `{{CHANGE_SLUG}}`. Do not fan out tasks; `/spec-tasks` does that after design is accepted.

### `.sdlc/changes/<slug>/specs/<capability>/delta.md`

Adapt `templates/delta.md`:
- Substitute `{{CAPABILITY}}` with the capability slug.
- In `## ADDED Requirements`, draft **exactly one** requirement reflecting the description:
  - Title: short noun phrase capturing the user-visible behavior or invariant.
  - Slug: prefix `req-` + dashed-lowercased-title.
  - `**Why:**` one line, usually mirrors `proposal.md`'s Why.
  - If the description is behavioral, write a `#### Scenario:` skeleton with `WHEN`/`THEN`, using `[[term-slug]]` markup for domain terms (define the terms in `## ADDED Terms` below).
  - If the description is an invariant, write a `#### Criteria` bullet.
- In `## ADDED Terms`: if any `[[term-slug]]` was introduced, draft a stub term entry with `**Definition:**` and `**Notes:**` (use `<!-- TODO -->` for parts you cannot fill confidently).
- Keep `## MODIFIED Requirements`, `## REMOVED Requirements`, `## MODIFIED Terms`, `## REMOVED Terms` as the template's empty placeholder blocks.

## Step 6 — Report

Print a structured summary:

```
Created .sdlc/changes/<slug>/
  proposal.md                         first draft (review TODOs)
  design.md                           skeleton (refine with /spec-design)
  tasks.md                            skeleton (refine with /spec-tasks)
  specs/<capability>/delta.md         1 ADDED requirement: <req-slug>

Next:
  1. Review proposal.md and refine with /spec-requirements (or edit directly).
  2. Once Why/Scope/Rollout are TODO-free and the delta is right, run /spec-design.
```

## Constraints

- Never write to `.sdlc/specs/`.
- Never overwrite an existing change folder.
- Initial delta has AT MOST ONE requirement and AT MOST a few terms; richer requirement sets come from `/spec-requirements`.
- Do not invent rollout details, design content, or task breakdowns at this stage — those phases have their own commands.
