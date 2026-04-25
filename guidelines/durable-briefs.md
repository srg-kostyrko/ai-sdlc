# Durable Briefs

Briefs (proposals, designs, ADRs, task descriptions) describe **what** the system should do, not **how** to implement it. The agent reading the brief explores fresh and makes its own implementation decisions. Write so the brief stays useful as files are renamed, refactored, or moved.

## Rules

| Rule | Detail |
|------|--------|
| **Durability** | Describe interfaces, types, behavioral contracts. Never file paths or line numbers. |
| **Behavioral** | State what the system should do, not implementation steps. |
| **Current → Desired** | Describe current behavior before desired behavior. |
| **Acceptance** | Each criterion independently verifiable and testable. |
| **Scope boundaries** | Explicitly state what is out of scope. |
| **Key interfaces** | Name types and contracts that need to change, with why. |

## Durable phrasing

**Do:**
- Describe interfaces, types, behavioral contracts.
- Name specific types or function signatures the agent should look for.
- Reference domain concepts and business rules.

**Don't:**
- Reference file paths — they go stale.
- Reference line numbers.
- Assume current implementation structure will remain.
- Write procedural steps ("open X, then change Y").

**Good:** "The `SkillConfig` type should accept an optional `schedule` field of type `CronExpression`."
**Bad:** "Open `src/types/skill.ts` and add a `schedule` field on line 42."

**Good:** "When a user runs `/triage` with no arguments, they should see a summary of issues needing attention."
**Bad:** "Add a switch statement in the main handler function."

## Current vs. desired behavior

Every brief establishes context by describing:

1. **Current behavior** — what happens now (for bugs: the broken behavior; for enhancements: the status quo).
2. **Desired behavior** — what should happen after the work, including edge cases and error conditions.

This framing lets readers understand intent without knowing history.

## Acceptance criteria

Each criterion is:

- Independently verifiable
- Concrete and testable
- Described as an observable outcome

**Good:** "POST /api/users with a valid payload returns 201 and creates a user record retrievable by GET /api/users/:id."
**Bad:** "User creation should work correctly."

## Scope boundaries

State what is **not** in scope. This prevents:

- Gold-plating with adjacent improvements.
- Assumptions about related features.
- Scope expansion during implementation.

A brief without explicit non-goals invites scope creep.
