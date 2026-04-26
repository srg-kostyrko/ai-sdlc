---
description: Start a new change proposal. Asks one question (the problem), derives a slug, and creates .sdlc/changes/<slug>/ with a proposal stub plus design/tasks skeletons. No capability delta is created — that happens lazily in `/ai-sdlc:spec-requirements` when the first requirement is added.
argument-hint: ["<one-line seed>"]
---

You are creating a new ai-sdlc change proposal.

User input: $ARGUMENTS

## Step 1 — Parse arguments

- `seed` (optional): the full `$ARGUMENTS`, surrounding quotes stripped. Used only as a phrasing hint in the interview; never written verbatim to disk.

No slug is taken from arguments. The slug is derived after the interview (Step 4) and reported back to the user; rename with `mv` if it's wrong.

## Step 2 — Pre-flight

- Verify `.sdlc/` exists at the working directory. If not: print `Run /ai-sdlc:sdlc-init first.` and stop.

(Slug-collision checks happen in Step 4, once a candidate exists.)

## Step 3 — Establish the motivation

Before asking, check whether the current conversation already contains enough to write `## Why`.

### Step 3.0 — Reuse conversation context when present

You have enough context to skip Q1 when **all** of these hold:
- The user (in this conversation, not just the seed) has named concrete pain, friction, or a gap — not just a desired solution.
- A reader who has not seen the conversation could understand *why* this change is needed from 2–3 sentences you can write now.
- Nothing material is ambiguous (which system, what's broken, why now).

If yes:
1. Draft `## Why` (2–3 sentences) from the conversation. Use the user's own language where possible; do not invent motivation they did not state.
2. Show the draft and ask: `Use this as the proposal's "Why", or refine?` Wait for confirmation or edits.
3. Once confirmed, treat the result as the answer and skip to Step 4. Run the classifier below on the confirmed text — if it reads as solution-shaped after all, fall through to Step 3a.

If conversation context is thin, ambiguous, or only contains a solution shape → ask Q1.

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

4. If the user rejects all candidates and still cannot articulate a motivation, stop and tell them `/ai-sdlc:spec-propose` needs a concrete motivation — do not seed `## Why` with a TODO.

## Step 4 — Derive change slug

Derive the slug from the confirmed motivation (Q1 answer or Step 3a selection):
- Distill the change into a 2–4 word noun phrase grounded in the motivation.
- Lowercase + dashes only. Must match `^[a-z0-9][a-z0-9-]*$`.
- Avoid generic prefixes (`add-`, `fix-`, `update-`) unless the change is genuinely of that shape and no more specific phrasing applies.

Run collision checks:
- `.sdlc/changes/<slug>/` must not exist.
- `.sdlc/changes/archive/` must not contain any folder ending with `-<slug>`.

If a collision is hit, append a short disambiguating suffix grounded in the motivation (e.g. `-v2`, or a more specific noun) and re-run the checks. No interactive confirmation — the slug is surfaced in Step 7's report, and the user can rename the folder with `mv` before any other command references it.

## Step 5 — Create the change folder

`mkdir -p .sdlc/changes/<slug>/` (the `-p` flag also creates the `.sdlc/changes/` parent if this is the project's first change).

(Do not create `specs/` or `decisions/` yet — both are seeded lazily: `specs/<capability>/` by `/ai-sdlc:spec-requirements` when the first requirement lands, `decisions/` when an ADR is drafted.)

## Step 6 — Write skeleton

Read each template from `${CLAUDE_PLUGIN_ROOT}/templates/` (this plugin's `templates/` directory; the path can be determined from the location of this command file — its parent's parent is the plugin root). Substitute `{{CHANGE_SLUG}}` with the slug.

### `.sdlc/changes/<slug>/proposal.md`

Adapt `templates/proposal.md`:
- `## Why`: write the answer to Q1 (lightly cleaned for prose flow; do not add motivation the user did not state).
- `## What Changes`, `## Scope`: leave the template's `<!-- TODO -->` markers untouched. `/ai-sdlc:spec-requirements` walks these next.

### `.sdlc/changes/<slug>/design.md`

Write the template **as-is**, only substituting `{{CHANGE_SLUG}}`. Design content is `/ai-sdlc:spec-design`'s job once the proposal is TODO-free.

### `.sdlc/changes/<slug>/tasks.md`

Write the template **as-is**, only substituting `{{CHANGE_SLUG}}`. `/ai-sdlc:spec-tasks` fans out tasks after design is accepted.

## Step 7 — Report

Print a structured summary:

```
Created .sdlc/changes/<slug>/
  proposal.md                         Why filled; What Changes / Scope TODO
  design.md                           skeleton
  tasks.md                            skeleton

Slug not what you wanted? Rename with `mv .sdlc/changes/<slug>/ .sdlc/changes/<new-slug>/` before running the next command.
Next: /ai-sdlc:spec-requirements <slug> to interview, draft requirements, and seed the first capability delta.
```

## Constraints

- Never write to `.sdlc/specs/`.
- Never overwrite an existing change folder.
- Slug is derived from the motivation in Step 4 — never taken from `$ARGUMENTS`, never interactively confirmed. The user renames with `mv` if the derived slug is wrong.
- Do not create `specs/` here. The first capability delta is seeded by `/ai-sdlc:spec-requirements`.
- `## Why` is always user-confirmed: either reused from conversation context and confirmed in Step 3.0, answered via Q1, or a candidate motivation the user accepted/edited in Step 3a. No fabricated motivation, no TODO.
- `## What Changes`, `## Scope`, design content, task breakdowns, capability decisions, and the first requirement are deferred to their own commands. Do not pre-fill them here.
