---
name: spec-aware
description: Use this skill when working in a project that contains a `.sdlc/specs/` directory and the user references a feature, requirement, capability slug, or domain term — load the relevant capability `spec.md`, `GLOSSARY.md`, and `CHANGELOG.md`, plus any in-flight deltas under `.sdlc/changes/*/specs/<capability>/`, so the response is grounded in current spec truth. Triggers on questions like "what does the X spec say about Y", "is there a requirement for Z", and on any code work touching files mapped to a bounded context. Do NOT activate when the project lacks `.sdlc/specs/`.
---

You are loading ai-sdlc spec context to ground the current response.

## When this skill applies

The project's working directory has `.sdlc/specs/` AND the user's request involves:
- A feature, requirement, capability name, or domain term (Title Case in prose, or opt-in `[[slug]]`).
- Modifying code that maps to a bounded context.
- Asking about behavior, invariants, or rationale.

Do NOT use this skill when:
- `.sdlc/specs/` does not exist (project does not use ai-sdlc).
- The work is unrelated to the spec'd domain (build tooling, CI, ops, dependency upgrades).

## Procedure

1. **Inventory.** Run `ls .sdlc/specs/` to list bounded contexts. If empty, no living spec exists yet — note this and stop.

2. **Identify relevant capability(ies).** From the user's request:
   - Direct match: capability slug appears in the request.
   - Indirect: domain terms or feature names map to a capability via the capability's `GLOSSARY.md` or scenario text.
   - When ambiguous, prefer reading multiple capabilities' `spec.md` headers (cheap) before committing to one.

3. **Load the living spec for each relevant capability.** Read:
   - `.sdlc/specs/<capability>/spec.md` — current requirements.
   - `.sdlc/specs/<capability>/GLOSSARY.md` — current term definitions.
   - `.sdlc/specs/<capability>/CHANGELOG.md` — recent changes (skim).

3a. **Load project-wide steering when relevant** — `.sdlc/steering/{product,structure,tech}.md`. Read selectively: `product.md` for goal/scope questions, `structure.md` when the user asks about layout or conventions, `tech.md` for stack/constraint questions. Skip files that don't exist or contain only template placeholder text (`[Brief description ...]`, `[Pattern Name]`, etc.) — those are unpopulated and have no grounding value.

4. **Check in-flight changes.** For each relevant capability, look for `.sdlc/changes/*/specs/<capability>/delta.md`. Note any ADDED/MODIFIED/REMOVED slugs; these are not yet living truth but are in-flight commitments.

5. **Respond grounded.** Answer the user's actual question, citing requirement slugs and ADR identifiers where they support the answer. Do not echo the spec verbatim unless asked — use it to inform.

6. **Surface contradictions.** If the user states something that conflicts with the living spec, name the conflict explicitly: which requirement, which scenario, where it lives. Do not silently override the spec.

## What this skill must not do

- Do not modify any spec, glossary, delta, or change file. This skill is read-only.
- Do not invoke other workflow commands (`/spec-propose`, `/spec-archive`, etc.) — surface that the user might want to run them, but let them decide.
- Do not summarize the entire spec when only a narrow part is relevant; respond to what was asked.
