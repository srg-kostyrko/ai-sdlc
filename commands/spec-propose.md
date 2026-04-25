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

Ask Q1 and wait for the answer. Then classify it before writing.

### Q1 — Problem

Ask: "What's broken or missing today, and why does it need to change now? 2–3 sentences."

If a seed was provided, prefix the question with `You said: "<seed>". Expand:` so the user can build on their original phrasing.

### Classify the answer

Decide whether the answer reads as a **problem** (names pain, friction, or a gap) or a **solution** (names a thing to add/build/change without stated pain).

Solution-shaped signals:
- Leads with verbs like "add", "build", "introduce", "implement", "refactor to", "migrate to"
- Names a concrete artifact (library, pattern, module) without naming what it fixes
- No pain language ("hard to", "duplicated", "breaks", "can't", "slow", "fragile", "blocks")

If problem-shaped → carry the answer to Step 6 as `## Why`.

If solution-shaped → Step 3a.

### Step 3a — Investigate and suggest motivations

The user named a solution but not the underlying motivation. Help them surface it rather than bouncing them.

1. Dispatch an Explore subagent (medium thoroughness) scoped to the user's answer + seed. Goal: find concrete pain in the codebase that this solution would plausibly address — tight coupling, duplicated wiring, test friction, awkward extension points, missing seams, etc. Require file paths and line numbers.

2. Present 2–3 candidate motivations grounded in that evidence:

   ```
   You named a solution, not a problem. Here's what the codebase suggests it might be solving:

   1. <one-sentence motivation>
      Evidence: <path:line>, <path:line>

   2. <one-sentence motivation>
      Evidence: <path:line>

   3. <one-sentence motivation>
      Evidence: <path:line>

   Pick a number to use as your motivation, edit one, or write your own.
   ```

3. The user's selection (or rewrite) becomes `## Why`. Never auto-pick — the user must endorse the motivation.

4. If the user rejects all candidates and still cannot articulate a motivation, stop and tell them `/spec-propose` needs a concrete motivation — do not seed `## Why` with a TODO.

## Step 4 — Propose change slug

Derive a candidate slug from the confirmed motivation (Q1 answer or Step 3a selection):
- Distill the change into a 2–4 word noun phrase grounded in the motivation.
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

Next: /spec-requirements to interview, draft requirements, and seed the first capability delta.
```

## Constraints

- Never write to `.sdlc/specs/`.
- Never overwrite an existing change folder.
- Slug is always derived and confirmed in Step 4 — never taken from `$ARGUMENTS`.
- Do not create `specs/` here. The first capability delta is seeded by `/spec-requirements`.
- `## Why` is always user-confirmed: either Q1 directly, or a candidate motivation the user accepted/edited in Step 3a. No fabricated motivation, no TODO.
- `## What Changes`, `## Scope`, `## Rollout`, design content, task breakdowns, capability decisions, and the first requirement are deferred to their own commands. Do not pre-fill them here.
