---
description: Start a new change proposal. Asks one question (the problem), derives a slug, and creates .sdlc/changes/<slug>/ with a proposal stub plus design/tasks skeletons. No capability delta is created — that happens lazily in `/spec-requirements` when the first requirement is added.
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

Ask one question and wait for the answer. Hold the answer in memory.

### Q1 — Problem

Ask: "What's broken or missing today, and why does it need to change now? 2–3 sentences."

If a seed was provided, prefix the question with `You said: "<seed>". Expand:` so the user can build on their original phrasing.

The answer is the only content that gets written into `proposal.md ## Why`. If the user cannot articulate a problem, stop and tell them `/spec-propose` needs a concrete motivation — do not seed `## Why` with a TODO.

## Step 4 — Propose change slug

Derive a candidate slug from the answer to Q1:
- Distill the change into a 2–4 word noun phrase grounded in Q1 (the problem statement).
- Lowercase + dashes only. Must match `^[a-z0-9][a-z0-9-]*$`.
- Avoid generic prefixes (`add-`, `fix-`, `update-`) unless the change is genuinely of that shape and no more specific phrasing applies.

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
```

(Do not create `specs/` or `decisions/` yet — both are seeded lazily: `specs/<capability>/` by `/spec-requirements` when the first requirement lands, `decisions/` when an ADR is drafted.)

## Step 6 — Write skeleton

Read each template from `${CLAUDE_PLUGIN_ROOT}/templates/` (this plugin's `templates/` directory; the path can be determined from the location of this command file — its parent's parent is the plugin root). Substitute `{{CHANGE_SLUG}}` with the slug.

### `.sdlc/changes/<slug>/proposal.md`

Adapt `templates/proposal.md`:
- `## Why`: write the answer to Q1 (lightly cleaned for prose flow; do not add motivation the user did not state).
- `## What Changes`, `## Scope`, `## Rollout`: leave the template's `<!-- TODO -->` markers untouched. `/spec-requirements` walks these next.

### `.sdlc/changes/<slug>/design.md`

Write the template **as-is**, only substituting `{{CHANGE_SLUG}}`. Design content is `/spec-design`'s job once the proposal is TODO-free.

### `.sdlc/changes/<slug>/tasks.md`

Write the template **as-is**, only substituting `{{CHANGE_SLUG}}`. `/spec-tasks` fans out tasks after design is accepted.

## Step 7 — Report

Print a structured summary:

```
Created .sdlc/changes/<slug>/
  proposal.md                         Why filled; What Changes / Scope / Rollout TODO
  design.md                           skeleton
  tasks.md                            skeleton

Next: /spec-requirements to walk the TODOs and seed the first capability + requirement.
```

## Constraints

- Never write to `.sdlc/specs/`.
- Never overwrite an existing change folder.
- Slug is always derived and confirmed in Step 4 — never taken from `$ARGUMENTS`.
- Do not create `specs/` here. The first capability delta is seeded by `/spec-requirements`.
- `## Why` is sourced from Q1 only — no fabricated motivation, no TODO.
- `## What Changes`, `## Scope`, `## Rollout`, design content, task breakdowns, capability decisions, and the first requirement are deferred to their own commands. Do not pre-fill them here.
