---
description: Start a new change proposal. Runs a 3-question interview, derives a slug, and creates .sdlc/changes/<slug>/ with a minimal proposal, design/tasks skeletons, and an initial delta.
argument-hint: ["<one-line seed>"]
---

You are creating a new ai-sdlc change proposal.

User input: $ARGUMENTS

## Step 1 — Parse arguments

- `seed` (optional): the full `$ARGUMENTS`, surrounding quotes stripped. Used only as a phrasing hint in the interview; never written verbatim to disk.

No slug is taken from arguments. The slug is derived after the interview (Step 4) and confirmed by the user.

## Step 2 — Pre-flight

- Verify `.sdlc/` exists at the working directory. If not: print `Run /sdlc-init first.` and stop.

(Slug-collision checks happen in Step 4, once a candidate exists.)

## Step 3 — Interview

Ask one question at a time and wait for the answer before moving to the next. Hold answers in memory.

### Q1 — Problem

Ask: "What's broken or missing today, and why does it need to change now? 2–3 sentences."

If a seed was provided, prefix the question with `You said: "<seed>". Expand:` so the user can build on their original phrasing.

The answer is the only content that gets written into `proposal.md ## Why`. If the user cannot articulate a problem, stop and tell them `/spec-propose` needs a concrete motivation — do not seed `## Why` with a TODO.

### Q2 — Capability

Look at the answer to Q1. Identify the most likely **bounded-context capability** affected (e.g. `auth`, `billing`, `notifications`, `search`).

Ask: `This looks like it touches \`<suggested>\`. Confirm, or name a different capability slug to seed the delta against.`

The capability slug must be lowercase-dashed. Do not create directories under `.sdlc/specs/` — capabilities are seeded lazily, only by `/spec-archive`.

### Q3 — First requirement

Ask: "Describe the smallest behavior or invariant this change must guarantee. Triggered behavior (when X happens, then Y) becomes a Scenario; an always-true rule (the system shall Z) becomes a Criteria."

From the answer, derive:
- Title: short noun phrase capturing the user-visible behavior or invariant.
- Slug: `req-` + dashed-lowercased-title.
- Shape: `#### Scenario:` with `WHEN`/`THEN` if behavioral; `#### Criteria` in EARS-Ubiquitous form (`The [System] shall [action]`, explicit subject) if invariant.
- Any `[[term-slug]]` references introduced — collect for `## ADDED Terms`.

If the answer is too vague to ground a single requirement, ask one follow-up. If still vague after the follow-up, stop and tell the user `/spec-propose` needs a concrete first requirement before the change folder is created.

## Step 4 — Propose change slug

Derive a candidate slug from the interview answers:
- Distill the change into a 2–4 word noun phrase. Prefer the Q3 requirement title; fall back to a phrase grounded in Q1 if the requirement is too narrow to represent the whole change.
- Lowercase + dashes only. Must match `^[a-z0-9][a-z0-9-]*$`.
- Avoid generic prefixes (`add-`, `fix-`, `update-`) unless the change is genuinely of that shape and no more specific phrasing applies.
- Do not include the capability slug as a prefix unless it makes the slug meaningfully clearer.

Run collision checks on the candidate:
- `.sdlc/changes/<candidate>/` must not exist.
- `.sdlc/changes/archive/` must not contain any folder ending with `-<candidate>`.

Present to the user:

```
Proposed slug: <candidate>
Accept, or name a different slug.
```

If the user supplies an override: validate against the regex and re-run the collision checks. If a collision is found (active or archived), report it and ask for another. Only proceed once a clean slug is confirmed.

## Step 5 — Create the change folder

```
.sdlc/changes/<slug>/
.sdlc/changes/<slug>/specs/<capability>/
```

(Do not create `decisions/` yet — created lazily when an ADR is drafted.)

## Step 6 — Write skeleton

Read each template from `${CLAUDE_PLUGIN_ROOT}/templates/` (this plugin's `templates/` directory; the path can be determined from the location of this command file — its parent's parent is the plugin root). Substitute `{{CHANGE_SLUG}}` with the slug and `{{CAPABILITY}}` with the chosen capability slug.

### `.sdlc/changes/<slug>/proposal.md`

Adapt `templates/proposal.md`:
- `## Why`: write the answer to Q1 (lightly cleaned for prose flow; do not add motivation the user did not state).
- `## What Changes`, `## Scope`, `## Rollout`: leave the template's `<!-- TODO -->` markers untouched. `/spec-requirements` walks these next.

### `.sdlc/changes/<slug>/design.md`

Write the template **as-is**, only substituting `{{CHANGE_SLUG}}`. Design content is `/spec-design`'s job once the proposal is TODO-free.

### `.sdlc/changes/<slug>/tasks.md`

Write the template **as-is**, only substituting `{{CHANGE_SLUG}}`. `/spec-tasks` fans out tasks after design is accepted.

### `.sdlc/changes/<slug>/specs/<capability>/delta.md`

Adapt `templates/delta.md`:
- Substitute `{{CAPABILITY}}` with the capability slug.
- `## ADDED Requirements`: one requirement built from Q3:
  - `### Requirement: <Title> {#req-slug}`
  - `**Why:**` one line, mirroring `proposal.md ## Why`.
  - The Scenario or Criteria block derived in Q3.
- `## ADDED Terms`: one stub entry per `[[term-slug]]` introduced in Q3, with `**Definition:**` and `**Notes:**` (use `<!-- TODO -->` for parts the user did not specify).
- Keep `## MODIFIED Requirements`, `## REMOVED Requirements`, `## MODIFIED Terms`, `## REMOVED Terms` as the template's empty placeholder blocks.

## Step 7 — Report

Print a structured summary:

```
Created .sdlc/changes/<slug>/
  proposal.md                         Why filled; What Changes / Scope / Rollout TODO
  design.md                           skeleton
  tasks.md                            skeleton
  specs/<capability>/delta.md         1 ADDED requirement: <req-slug>

Next: /spec-requirements to walk the TODOs and audit the delta.
```

## Constraints

- Never write to `.sdlc/specs/`.
- Never overwrite an existing change folder.
- Slug is always derived and confirmed in Step 4 — never taken from `$ARGUMENTS`.
- Initial delta has exactly one requirement and at most a few stub terms.
- `## Why` is sourced from Q1 only — no fabricated motivation, no TODO.
- `## What Changes`, `## Scope`, `## Rollout`, design content, and task breakdowns are deferred to their own commands. Do not pre-fill them here.
