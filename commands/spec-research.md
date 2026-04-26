---
description: Investigate a loose idea before committing to a change. Runs codebase + (optional) web research via subagents, drafts approaches with tradeoffs, runs a viability check and grill-me, writes `.sdlc/research/<slug>.md`. Concludes with Proceed / Drop / Extend recommendation; never auto-creates a change folder.
argument-hint: ["<topic seed>"]
---

You are running pre-change research for an idea the user wants to investigate before committing to a proposal.

The flow is **frame → scan → explore (subagents) → draft approaches → viability check → grill-me → finalize**. The artifact is a single file `.sdlc/research/<slug>.md`. A change folder is **not** created here — the research either recommends one (`/ai-sdlc:spec-propose` next) or recommends against.

## Step 1 — Parse arguments

User input: $ARGUMENTS

- `seed` (optional): full `$ARGUMENTS`, surrounding quotes stripped. Used as a phrasing hint only.

## Step 2 — Pre-flight

- Verify `.sdlc/` exists. If not: print `Run /ai-sdlc:sdlc-init first.` and stop.
- Ensure `.sdlc/research/` exists (`mkdir -p`). It may not have been created by older `/ai-sdlc:sdlc-init` runs.

## Step 3 — Frame the question

Before searching anything, get a one-line research question on the table.

If the conversation already names a concrete question (not just a vague topic), skip Q1: draft the question in 1–2 sentences, show it, ask "Use this as the research question, or refine?" Once confirmed, carry it forward.

Otherwise ask **Q1**: "What do you want to know? One question, in your own words. (e.g. 'should we replace X with Y?', 'what would it take to support Z?', 'is library W viable for our stack?')"

If a seed was provided, prefix with `You said: "<seed>". Sharpen into a question:`.

## Step 4 — Lightweight scan

Gather only the metadata needed to know what context already exists. Do NOT read full file contents yet.

- `.sdlc/changes/` (excluding `archive/`): list active change slugs. Note any whose name overlaps the research question — they may be the right home for findings.
- `.sdlc/research/`: list existing research files. If one looks like a duplicate of the new question, surface it and ask whether to extend that file rather than start fresh.
- `.sdlc/steering/*.md` (if any): note which exist. Read `product.md` and `tech.md` if present — they shape what counts as a viable approach.
- `.sdlc/specs/`: list existing capability folders for context on what bounded contexts already exist.

Keep total content loaded under ~300 lines. The subagents handle the heavy lifting.

## Step 5 — Decide what to explore

Based on the question and scan, decide which subagents to dispatch. Skip a dispatch if the answer is obvious from context already loaded.

- **Codebase exploration** (almost always): dispatch `Explore` subagent (medium thoroughness) scoped to the question. Prompt should ask for: relevant files with paths and line numbers, current patterns in the area, existing seams or extension points, friction the user named (if any).
- **Web research** (only when the question names an external technology, library, standard, or pattern the codebase can't answer alone): dispatch `general-purpose` subagent. Prompt should ask for: official-docs summary, latest stable version, known gotchas, license, last-release recency. Require links.

Dispatch both in parallel when both apply. Each must return a summary, **not** raw output. Cap each summary at ~200 lines.

If neither subagent is needed (small question fully answerable from loaded context), say so and proceed.

## Step 6 — Draft approaches in memory

Synthesize 2–3 candidate approaches. Hold all drafting in memory; do not write to disk until Step 10.

For each approach:
- **Name**: noun phrase, 2–4 words.
- **How it works**: 2–3 sentences grounded in the subagent findings.
- **Pros / Cons**: concrete, citing file:line or source URL where applicable.
- **Scope**: small / medium / large rough estimate.

If the question genuinely admits only one viable approach (rare), draft the single approach and note explicitly *why* alternatives are ruled out.

Present the approaches to the user. Recommend one and explain why in 1–2 sentences.

## Step 7 — Viability check (mandatory once an approach is picked)

After the user selects an approach (or accepts the recommendation), dispatch a subagent to verify viability before recording it.

Prompt template: "Verify viability of [approach name] for this project. Check: (1) are the named technologies/libraries actively maintained (last release within 12 months)? (2) license compatibility with the existing stack? (3) do the components actually compose for [the question's use case]? (4) any known showstoppers — critical bugs, security CVEs, platform limits? Return only issues found, or 'No issues found'."

If issues surface, present them and revisit Step 6. If clean, proceed.

## Step 8 — Grill-me (mandatory)

Run the `grill-me` skill against the in-memory recommendation. Walk each branch:

- **Premise** — does the research question itself still hold given findings, or did exploration reveal the real question is different?
- **Approach pressure** — what would force you off the recommended approach? At what scale, cost, or constraint does it break?
- **Hidden cost** — what does the recommendation quietly require (migration, training, ops, vendor lock-in) that the Pros/Cons didn't surface?
- **Reversibility** — if this turns out wrong six months in, what's the cost of backing out?
- **No-build option** — has "do nothing" been honestly compared? What's the cost of not doing this?
- **Wrong-bin risk** — is this actually a Drop or Extend (existing change) wearing Proceed's clothes?

Question count is open — keep going within a branch until it resolves. Apply each agreed change to the in-memory draft.

Before concluding, ask: "Anything else to challenge before we record this?"

## Step 9 — Classify the recommendation

Land on exactly one:

- **Proceed** — research supports a new change. Draft a 2–3 sentence Why suitable for seeding `/ai-sdlc:spec-propose`. The user runs `/ai-sdlc:spec-propose` next; in-session conversation context carries the Why automatically.
- **Extend existing change** — findings belong to an active change, not a new one. Name the slug and what to add. The user runs `/ai-sdlc:spec-requirements <slug>` next.
- **Drop** — recommendation is don't build this. State the disqualifier plainly. The artifact persists as a record of what was evaluated and why it was rejected.

## Step 10 — Derive slug and write

Derive a research slug from the confirmed question:
- 2–4 word noun phrase grounded in the question (not the recommendation — slug should still make sense if recommendation is Drop).
- Lowercase + dashes only. Must match `^[a-z0-9][a-z0-9-]*$`.

Collision check:
- `.sdlc/research/<candidate>.md` must not already exist. If it does, ask the user whether to overwrite or pick a different slug.

Read `${CLAUDE_PLUGIN_ROOT}/templates/research.md`. Substitute `{{RESEARCH_SLUG}}`. Fill all sections from the in-memory draft:
- `## Question` — the confirmed research question.
- `## Findings` — bulleted summary from subagent outputs, each with file:line or URL.
- `## Approaches Considered` — every approach drafted in Step 6, including ones the user rejected (record kept for future readers).
- `## Viability Notes` — Step 7 output verbatim or condensed.
- `## Recommendation` — Proceed / Drop / Extend, with the paragraph from Step 9.
- `## Next Step` — the concrete next command, or "no further action" for Drop.

Write to `.sdlc/research/<slug>.md`.

## Step 11 — Report

Print:

```
Wrote .sdlc/research/<slug>.md
Recommendation: <Proceed | Drop | Extend existing change>
Next: <the next-step line from the artifact>
```

## Constraints

- Never create `.sdlc/changes/<slug>/` from this command. That is `/ai-sdlc:spec-propose`'s job.
- Never modify existing files under `.sdlc/changes/`, `.sdlc/specs/`, or other research files.
- The artifact persists even on Drop — research records exist so the same question doesn't get re-investigated cold later.
- Subagent raw output never enters main context — only summaries.
- One question per research run. If exploration surfaces a second distinct question, finish this one and recommend a follow-up `/ai-sdlc:spec-research` run for the second.

## Error scenarios

- **Missing `.sdlc/`.** Stop, suggest `/ai-sdlc:sdlc-init`.
- **Slug collision with existing research.** Ask the user: overwrite, pick a new slug, or extend the existing file in a fresh session.
- **Subagent returns nothing useful.** Surface to the user — do not invent findings. Offer to retry with a sharper subagent prompt or proceed with what's available.
- **Grill-me reaches no shared understanding.** Stop and write nothing; report the open question.
- **User cannot pick an approach after viability check.** Stop and write nothing; suggest re-running with a narrower question.
