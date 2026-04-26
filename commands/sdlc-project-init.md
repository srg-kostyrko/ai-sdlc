---
description: Bootstrap a greenfield project through a guided interview. Produces .sdlc/steering/product.md and tech.md so commands have project-wide grounding from day one.
argument-hint: ["<short project description>"]
---

You are bootstrapping a greenfield project's steering files via interview.

User input: $ARGUMENTS

## Step 1 — Greenfield guard rails

Run two checks before starting the interview. If either fails, stop with the indicated message.

**Check 1 — already initialized.**
Read `.sdlc/steering/product.md`. If it exists and contains real content beyond the template's placeholder text (e.g. has filled sections, not just `[Brief description ...]`):

> Steering already exists. Use `/ai-sdlc:sdlc-steering` to update.

Stop.

**Check 2 — brownfield detection.**
Glob for source files: `**/*.{ts,tsx,js,jsx,py,go,rs,java,rb,cs,kt,swift,ex,erl,clj,scala,php,c,cpp,h,hpp}`. Config files alone (`package.json`, `Dockerfile`, `.gitignore`, `pyproject.toml`, `Cargo.toml`, `go.mod`, etc.) do NOT count — people scaffold these before writing code.

If source files are found:

> This looks like a brownfield project — use `/ai-sdlc:sdlc-steering` instead.

Stop.

If both checks pass, proceed to the interview.

## Step 2 — Interview

Ask questions **one at a time**. Wait for the user's answer before asking the next.

If `$ARGUMENTS` provides a project description, use it to skip or pre-fill obvious questions.

Throughout all phases:
- **Silently track domain terms** the user mentions. Watch for domain-specific nouns, concepts with specific meaning in this project, terms that could be ambiguous. (These will surface as a quick review at the end and seed initial per-capability glossaries when changes are proposed.)
- **Silently watch for hard-to-reverse technical decisions** (database choice, monorepo vs polyrepo, language choice, deployment platform). These will be surfaced as ADR candidates at the end.

### Phase 1 — Product (4–5 questions)

1. What are you building? (one sentence)
2. Who is it for? (target users / audience)
3. What are the 3–5 core capabilities?
4. What problem does this solve that existing tools don't? (value proposition)
5. Any known constraints? (regulatory, platform, timeline)

Skip question 5 if the user says "none" or the answer is obvious from prior context.

### Phase 2 — Technical (adaptive, 2–3 questions)

Adapt based on the project type revealed in Phase 1.

1. What's your primary language? (skip if obvious from Phase 1)
2. Project-type-specific follow-ups — ask only the relevant branch:
   - **Library/SDK**: Distribution method (npm, PyPI, crates.io, ...)? Public or internal?
   - **CLI tool**: Target platforms? Runtime dependencies?
   - **Web app / API**: Framework? Data storage? Deployment target?
   - **Agent / AI system**: Orchestration approach? Model provider(s)?
   - **Other**: Key technical building blocks?
3. External integrations or dependencies?

### Phase 3 — Wrap-up

**Domain term review.** Present the terms you tracked:

> "I picked up these domain terms — anything to add, remove, or clarify?"

List each with a proposed one-line definition. Accept corrections. (Do NOT write a glossary file — terms will land in per-capability `GLOSSARY.md` when capabilities are introduced via change proposals.)

**ADR candidates (conditional).** If hard-to-reverse decisions surfaced:

> "These look like architectural decisions worth recording. Run a one-shot decision capture for each when you're ready (or let `/ai-sdlc:spec-design` draft them when relevant)."

List candidates. Skip silently if none surfaced.

## Step 3 — Generate artifacts

`mkdir -p .sdlc/steering/`, then generate both files in one pass.

### `.sdlc/steering/product.md`

Read `${CLAUDE_PLUGIN_ROOT}/templates/steering/product.md`. Fill it from Phase 1 answers:

- **Brief description** (intro paragraph): from question 1, lightly framed by question 2.
- **Core Capabilities**: bullet list from question 3.
- **Target Use Cases**: derived from questions 1 + 2.
- **Value Proposition**: from question 4.
- **Constraints**: include the section only if the user mentioned constraints in question 5; otherwise omit the section entirely.

Write to `.sdlc/steering/product.md`.

### `.sdlc/steering/tech.md`

Read `${CLAUDE_PLUGIN_ROOT}/templates/steering/tech.md`. Fill it from Phase 2 answers.

**Conditional sections by project type** — include only what's relevant:
- A CLI tool does not need *Deployment* or *Database* sections (drop them).
- A library does not need *Database* or *Deployment* sections.
- A web app gets the full template.
- An agent/AI system gets *Orchestration* in place of *Framework*.

**Common commands**: leave `TBD` placeholders since no code exists yet.

Write to `.sdlc/steering/tech.md`.

### `.sdlc/steering/structure.md`

**Do not generate**. Structure depends on real code organization; defer to `/ai-sdlc:sdlc-steering` after code exists.

## Step 4 — Report

Print:

```
Created:
  .sdlc/steering/product.md
  .sdlc/steering/tech.md

Deferred:
  .sdlc/steering/structure.md  (run /ai-sdlc:sdlc-steering once you have code)
```

If ADR candidates were flagged in Phase 3, list them again under `ADR candidates:`.

End with:

> Run /ai-sdlc:spec-propose "<one-line seed>" to start your first change.

## Constraints

- Never include keys, passwords, or secrets in steering files.
- When in doubt, add rather than replace.
- Avoid documenting agent-specific directories (`.claude/`, `.cursor/`, etc.).
- Light references to `.sdlc/specs/` and `.sdlc/steering/` are acceptable.
- One question at a time during the interview.
