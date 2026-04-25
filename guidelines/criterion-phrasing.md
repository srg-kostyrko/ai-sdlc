# Criterion Phrasing — EARS-Ubiquitous Form

In ai-sdlc requirements:
- **Behavioral scenarios** use Gherkin-style `WHEN ... THEN ...` blocks.
- **Invariant criteria** use **EARS-Ubiquitous form**: `The [System] shall [action]`.

This guideline is about the *invariant criteria* form (the `#### Criteria` bullets).

## The form

```
The [System] shall [action]
```

- **Subject** — explicit named system or service. Not `it`, not `the system` generically — the actual name.
- **Verb** — `shall` (mandatory). Avoid `should`, `must`, `will`, `is` — those degrade enforceability.
- **Action** — concrete, testable obligation.

## Why EARS for invariants

Scenarios describe behaviors triggered by events. Invariants describe properties that hold *always*, independent of any trigger. Plain prose makes them easy to write vaguely. EARS-Ubiquitous form forces:

- An explicit subject — you can't write a passive invariant.
- A `shall` obligation — the rule becomes contractual, not aspirational.
- A specific action — vague invariants fail the form check.

## Examples

**Good:**
- The Auth Service shall persist TOTP secrets such that compromise of the data store does not reveal the plaintext.
- The Auth Service shall persist recovery codes in a form from which the original code cannot be recovered.
- The Audit Logger shall persist every successful authentication within 100ms.
- The Notifications Service shall reject emails to addresses without a verified domain.

**Bad — passive voice (no subject):**
- ❌ Recovery codes are hashed.
- ❌ TOTP secrets are encrypted at rest.
- ❌ Audit logs are persisted quickly.

→ Restate with explicit subject + shall:
- ✅ The Auth Service shall persist recovery codes in a form from which the original code cannot be recovered.
- ✅ The Auth Service shall persist TOTP secrets such that compromise of the data store does not reveal the plaintext.
- ✅ The Audit Logger shall persist authentication events within 100ms.

**Bad — vague verbs (no concrete action):**
- ❌ The Auth Service shall handle TOTP correctly.
- ❌ The Notifications Service shall support multiple delivery channels.

→ Specify the concrete obligation:
- ✅ The Auth Service shall reject TOTP codes more than 30 seconds old.
- ✅ The Notifications Service shall deliver via email and SMS, in that order.

**Bad — soft verbs:**
- ❌ The Auth Service should hash recovery codes. (`should` is recommendation, not obligation)
- ❌ The Auth Service will hash recovery codes. (`will` is prediction, not contract)

→ Always `shall` for invariants.

## What, not how — external observability

Phrase the `[action]` as a property verifiable from the subject's external boundary: observable behavior, externally visible state, or a published contract. Library choice, algorithm name, internal schema field, and internal method name belong in `design.md`.

**Carve-out — public contracts.** When the change's purpose is to expose a contract to outside callers (a new HTTP endpoint, wire format, CLI surface, SDK signature), the contract shape *is* the requirement. Name the path, payload, or signature directly — that's still externally observable.

**Bad — names the implementation:**
- ❌ The Auth Service shall hash recovery codes using bcrypt with cost factor 12. (bcrypt + cost factor are *how*)
- ❌ The Billing Service shall store invoices in the `invoices_v2` table with a `tenant_id` foreign key. (table + column names are *how*)
- ❌ The Notifications Service shall call `EmailGateway.send()` for each delivery. (internal method name is *how*)

→ Restate as observable property, or move to `design.md`:
- ✅ The Auth Service shall persist recovery codes in a form from which the original code cannot be recovered.
- ✅ The Billing Service shall associate every persisted invoice with exactly one tenant.
- ✅ The Notifications Service shall deliver each accepted notification at least once.

**Good — public contract carve-out:**
- ✅ The Auth Service shall accept `POST /sessions` with a JSON body `{username, password}` and respond with `200` plus a session token on success. (HTTP endpoint is the published contract this change exposes)
- ✅ The CLI shall accept `auth login --username <u>` and exit `0` on successful authentication. (CLI surface is what callers depend on)

## Quick check

Before writing a criterion, ask:

1. Does it name a specific system or service? If no, add one.
2. Is the verb `shall`? If no, replace.
3. Is the action concrete enough to test? If no, sharpen.
4. Is the action verifiable from the subject's external boundary, without reading its internals? If no, restate as an observable property — or, if it names a public contract this change exposes, leave it.

If all four pass, the criterion is in form.
