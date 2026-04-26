# {{CHANGE_SLUG}} — Design

<!-- Required sections (gate-checked): Approach, Goals / Non-Goals, File Structure Plan, Risks. -->
<!-- Reference requirement slugs from this change's deltas (e.g. req-totp-enrollment). -->

## Approach

<!-- TODO: how the change is built. Components, control flow, data shapes. -->

## Goals / Non-Goals

**Goals:**
- <!-- TODO: primary objective(s) of this change. -->

**Non-Goals:**
- <!-- TODO: explicitly excluded scope; future considerations; integration points deferred. -->

## File Structure Plan

<!-- TODO: files to create or modify, grouped by component. Concrete paths, not abstractions. -->

## Risks

<!-- TODO: what could go wrong. Migration concerns, perf budgets, security implications, observability gaps. -->

## Decisions

<!-- Optional. Create an ADR only if the decision meets ALL three criteria: -->
<!--   (1) hard to reverse, (2) surprising without context, (3) result of a real trade-off. -->
<!-- Most changes have zero ADRs; that is normal. -->

<!--
What qualifies (from skills/domain-model/ADR-FORMAT.md):
- Architectural shape: "we use a monorepo"; "write model is event-sourced, read model projected to Postgres".
- Integration patterns between contexts: "Ordering and Billing communicate via domain events, not synchronous HTTP".
- Technology choices that carry lock-in: database, message bus, auth provider, deployment target.
  Not every library — only the ones that would take a quarter to swap out.
- Boundary and scope decisions: "Customer data is owned by the Customer context; others reference by ID only".
  The explicit no-s are as valuable as the yes-s.
- Deliberate deviations from the obvious path: "manual SQL instead of an ORM because X".
  Anything where a reasonable reader would assume the opposite — these stop the next engineer from "fixing" something deliberate.
- Constraints not visible in the code: compliance, partner SLAs, legal limitations.
- Rejected alternatives when the rejection is non-obvious — otherwise someone will suggest it again in 6 months.

If a decision qualifies: draft at changes/<slug>/decisions/draft-<name>.md and reference it below.
-->

<!--
Optional sections — add when relevant; not gate-required:

## Data Models
For changes that touch persistence. Capture only what changes: domain model deltas, schema additions, key/index decisions.

## Testing Strategy
For changes warranting explicit test design beyond the per-task validation rows: integration scope, e2e flows, perf/load budgets.

## Components and Interfaces
For changes that introduce new components or significantly reshape existing contracts. Include only when interfaces are non-obvious from the deltas.

## Migration Strategy
For changes that move data, change schemas, or stage rollout beyond a feature flag. Phases, rollback triggers, validation checkpoints.
-->

