---
name: spec-scenario-rewrite
description: Use this skill when the user asks to convert prose acceptance criteria into structured `WHEN`/`THEN` scenarios for an ai-sdlc requirement, with phrasings like "rewrite this as a scenario", "convert to WHEN/THEN", "structure this as Gherkin", or "make this a scenario". Returns properly formatted `#### Scenario:` blocks with `[[term-slug]]` markup; declines to force the format on invariant statements that should be `#### Criteria` instead.
---

You are converting prose into structured scenario format.

## When this skill applies

User explicitly asks for scenario formatting. Typical phrasings include "rewrite as WHEN/THEN", "structure as scenario", "convert to Gherkin", "make this a scenario".

## Procedure

1. **Read the prose.** Identify the user's input — a requirement description, an acceptance bullet, a paragraph of prose.

2. **Decide: scenario or criterion?**
   A scenario has shape: trigger → outcome (a verb that *happens*, an observable state change).
   A criterion is invariant: a property that holds independent of any trigger ("X is always Y", "X must be encrypted").
   - If the prose has a clear trigger and outcome → produce a `#### Scenario:` block.
   - If the prose is invariant → tell the user this looks like an invariant; suggest `#### Criteria` bullet instead. Do not force WHEN/THEN onto invariants — the structure carries no meaning there.

3. **For scenarios, extract clauses:**
   - **WHEN:** the trigger — a user action, a system event, a time, a request. Multiple WHENs allowed for compound preconditions; chain with `AND`.
   - **THEN:** the primary observable outcome.
   - **AND (after THEN):** secondary outcomes — state changes, side effects, audit log entries, emitted events.

4. **Apply term markup.** Every domain-specific noun phrase becomes `[[term-slug]]`. If a term isn't in glossary, note it (the user can run `/spec-requirements` to add to `## ADDED Terms`, or invoke the `spec-glossary-suggest` skill).

5. **Produce the scenario:**
   ```
   #### Scenario: <short imperative name describing the situation>
   - WHEN [[user]] does X
   - AND <additional precondition>
   - THEN <observable outcome>
   - AND <secondary outcome>
   ```

   - Scenario name: short and descriptive (e.g. "Admin enrolls TOTP successfully", not "TestEnrollment1"). Sentence case, no trailing punctuation.
   - Each WHEN/AND/THEN bullet is one short sentence.
   - 2–6 clauses total. If more, the scenario probably bundles two scenarios — split.

6. **For criteria, output instead:**
   ```
   #### Criteria
   - <invariant statement>
   ```

## What this skill must not do

- Do not invent preconditions or outcomes the prose did not state. If something is missing, ask.
- Do not force WHEN/THEN format onto invariants; degrade gracefully to `#### Criteria`.
- Do not introduce new terms without flagging — wrap as `[[term-slug]]` and surface to the user.
- Do not modify files; return the formatted block as a response so the user can paste it into the right delta.
