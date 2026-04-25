---
name: spec-lookup
description: Use this skill when the user explicitly wants to locate a requirement, term, scenario, or ADR by keyword in a `.sdlc/`-organized project. Triggers on phrasings like "find the requirement about X", "what's the slug for Y", "look up the ADR on Z", "where do we say [behavior]". Returns matching slugs with source paths and short snippets, including matches in in-flight deltas.
---

You are searching the ai-sdlc spec tree for an item the user wants to find.

## When this skill applies

User explicitly asks to look up, find, or locate something in the spec tree. Keywords typically include "find", "where", "slug", "look up", "search". The project must have `.sdlc/specs/` or `.sdlc/changes/` present.

## Procedure

1. **Classify the query.** Determine which of these the user is searching for:
   - Requirements (heads `### Requirement:`, slugs `{#req-...}`)
   - Terms (heads `### Term:`, slugs `{#term-...}`)
   - Scenarios (heads `#### Scenario:`)
   - ADRs (files matching `[0-9]{4}-*.md` under `.sdlc/specs/*/decisions/` or `.sdlc/decisions/`)
   - Generic — any of the above (run all four searches and group results)

2. **Search living spec.** Use grep across the appropriate paths:
   - Requirements/scenarios: `.sdlc/specs/*/spec.md`
   - Terms: `.sdlc/specs/*/GLOSSARY.md`
   - ADRs: `.sdlc/specs/*/decisions/*.md` and `.sdlc/decisions/*.md`

3. **Search in-flight deltas.** Same patterns under `.sdlc/changes/*/specs/*/delta.md`. Tag results from this scope as `(in-flight: <change-slug>)`.

4. **Rank and de-dupe.** If a slug appears in both living spec and a delta:
   - ADDED in delta but not in living: in-flight only.
   - MODIFIED in delta and present in living: show both, label which is current truth.
   - REMOVED in delta and present in living: living version with a flag.

5. **Format the result.** For each match:
   ```
   <slug>  —  <path>:<line>
     <short snippet, ≤120 chars>
   ```
   Group by category (Requirements / Terms / Scenarios / ADRs).

6. **Empty result.** If nothing matches, suggest the closest slugs (prefix or substring match against all known slugs) before giving up.

## What this skill must not do

- Do not modify any file.
- Do not summarize the matches' contents at length — return locators, not exegesis.
- Do not silently skip in-flight matches; users often need to know about pending changes.
