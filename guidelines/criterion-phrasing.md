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
- The Auth Service shall store TOTP secrets encrypted with the per-tenant KMS key.
- The Auth Service shall hash recovery codes using bcrypt with cost factor 12.
- The Audit Logger shall persist every successful authentication within 100ms.
- The Notifications Service shall reject emails to addresses without a verified domain.

**Bad — passive voice (no subject):**
- ❌ Recovery codes are hashed.
- ❌ TOTP secrets are encrypted at rest.
- ❌ Audit logs are persisted quickly.

→ Restate with explicit subject + shall:
- ✅ The Auth Service shall hash recovery codes.
- ✅ The Auth Service shall store TOTP secrets encrypted with the per-tenant KMS key.
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

## Quick check

Before writing a criterion, ask:

1. Does it name a specific system or service? If no, add one.
2. Is the verb `shall`? If no, replace.
3. Is the action concrete enough to test? If no, sharpen.

If all three pass, the criterion is in form.
