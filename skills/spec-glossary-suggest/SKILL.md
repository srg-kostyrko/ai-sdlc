---
name: spec-glossary-suggest
description: Use this skill when authoring or reviewing requirement scenarios in an ai-sdlc delta to flag domain-specific noun phrases that should be defined in the capability's `GLOSSARY.md`. Triggers when the user asks "what terms am I missing?", "which terms need glossary entries?", or after a `/spec-requirements` pass when domain vocabulary is in flux. Surfaces unresolved `[[term-slug]]` references and unwrapped capitalized noun phrases as candidates; never modifies files autonomously.
---

You are auditing requirement text for missing glossary coverage.

## When this skill applies

User asks for term-coverage review of:
- A specific delta file (e.g. `.sdlc/changes/<slug>/specs/<cap>/delta.md`)
- A specific living capability spec
- All in-flight deltas in a change

Project must contain `.sdlc/specs/` or `.sdlc/changes/`.

## Procedure

1. **Identify scope.** From user input or context, determine which file(s) to audit. If unclear, list the candidates and ask.

2. **Identify the target glossary.** Each delta or spec belongs to a capability. The relevant glossary is `.sdlc/specs/<capability>/GLOSSARY.md` (the living glossary), plus any `## ADDED Terms` entries in the same change's deltas (in-flight terms).

3. **Extract candidate terms** from scenarios and criteria in scope:
   - **Explicit references:** every `[[term-slug]]` markup. Confirm each resolves to a glossary entry (living or in-flight); flag unresolved ones.
   - **Implicit candidates:** capitalized noun phrases inside `WHEN`/`THEN`/criterion text that are NOT obviously generic English (e.g. "User", "Session", "Recovery Code") AND are NOT proper nouns of external systems (e.g. "Postgres", "GitHub", "Stripe"). Heuristic — prone to false positives; surface as candidates, not assertions.

4. **Classify each candidate.** Three buckets:
   - **Unresolved `[[term]]`:** the markup is wrong (typo) or the term needs to be added to `## ADDED Terms` in the change's delta.
   - **Implicit candidate (likely term):** capitalized phrase that pattern-matches a domain concept; suggest wrapping as `[[term-slug]]` and adding a glossary entry.
   - **Implicit candidate (likely not):** capitalized phrase that's probably a proper noun or sentence-start. Surface but mark low-confidence.

5. **Report findings.** Group by capability and severity:
   ```
   Capability: auth
     Unresolved [[principal]] (high) — auth/delta.md:23
       Define in: changes/<slug>/specs/auth/delta.md ## ADDED Terms

     Capitalized "Recovery Code" — auth/delta.md:31  (medium)
       Suggest: wrap as [[recovery-code]] and add to ## ADDED Terms

     Capitalized "Postgres" — auth/delta.md:47  (low — likely proper noun)
       Action: probably none.
   ```

## What this skill must not do

- Do not modify any file. This skill is read-only by design — the user (or `/spec-requirements`) is the one who decides whether each candidate is a real term.
- Do not invent definitions for the suggested terms; surface the candidate, leave authoring to the user.
- Do not flag every capitalized word — apply the heuristic and accept some false negatives in exchange for low noise.
