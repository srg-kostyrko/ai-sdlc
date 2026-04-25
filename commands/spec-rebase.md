---
description: Reconcile an active change's deltas with the current living spec when /spec-archive aborted on slug conflicts.
argument-hint: [<slug>]
---

You are rebasing a change's deltas against the current living spec. Use this after `/spec-archive` aborted on slug conflicts (typically because another change was archived first and moved your baseline).

## Step 1 — Resolve the change

User input: $ARGUMENTS

- If `$ARGUMENTS` contains a slug, use `.sdlc/changes/<slug>/`. If that folder doesn't exist, list active changes and ask which one.
- If `$ARGUMENTS` is empty:
  - If exactly one folder exists under `.sdlc/changes/` (excluding `archive/`), use it.
  - Otherwise list the active changes and ask which one.

## Step 2 — Read state

For each `.sdlc/changes/<slug>/specs/<capability>/delta.md`:

- Read the delta.
- Read the corresponding living `.sdlc/specs/<capability>/spec.md` and `GLOSSARY.md` (if they exist).

## Step 3 — Detect conflicts

Run the same checks `/spec-archive` runs in its sanity step:

For each delta block, classify each entry:

| Block in delta | Status in living spec | Conflict? |
|---|---|---|
| ADDED `req-foo` | does not exist | clean |
| ADDED `req-foo` | already exists | **conflict A**: someone else added it |
| MODIFIED `req-foo` | exists | clean |
| MODIFIED `req-foo` | does not exist | **conflict B**: target was removed |
| REMOVED `req-foo` | exists | clean |
| REMOVED `req-foo` | does not exist | **conflict C**: already removed |

Same matrix for Terms.

## Step 4 — Resolve, one entry at a time

For each conflict, walk the user through the choice:

**Conflict A (ADDED, but already exists)**
- Show: the delta's ADDED block AND the existing living version.
- Options:
  - Move to MODIFIED (your version replaces theirs).
  - Drop from your delta (their version stands; your scenarios may need to merge into theirs — handle separately).
  - Rename your slug (rare; only if your requirement is genuinely different).

**Conflict B (MODIFIED, but missing)**
- The target requirement was removed by an earlier-archived change.
- Options:
  - Move your block to ADDED (re-introduce the requirement, possibly under a new slug).
  - Drop entirely (your modification is no longer relevant).

**Conflict C (REMOVED, but already gone)**
- Drop the REMOVED entry. Already done.

## Step 5 — Apply resolutions

For each user choice, modify the appropriate `delta.md` file in place. Do not modify the living spec — the goal is to make your deltas applicable, not to bypass `/spec-archive`'s merge.

After each resolution, re-run the conflict matrix against the updated delta to confirm progress.

## Step 6 — Validation impact

Some resolutions invalidate validation rows (e.g. a MODIFIED→ADDED move changes the row category). After resolution, suggest re-running `/spec-validate` to refresh the skeleton. Existing evidence is preserved by `/spec-validate`'s merge.

## Step 7 — Report

Print a structured summary:

```
Rebase complete for <slug>:
  Conflicts resolved:
    specs/auth/delta.md: req-totp-enrollment  ADDED → MODIFIED
    specs/auth/delta.md: req-old-totp        MODIFIED → ADDED (new slug: req-old-totp-v2)
  Validation rows affected: 3 (run /spec-validate to refresh)

Re-run /spec-archive when ready.
```

If any conflicts remain unresolved (user deferred the choice), list them and stop without claiming success.

## Constraints

- Never modify living `spec.md` or `GLOSSARY.md` — rebase only edits your in-flight deltas.
- Slug rules still apply: changing a slug means REMOVED + ADDED with the new slug, never an in-place rename.
- Do not paraphrase requirement bodies; preserve verbatim unless the user explicitly chooses to rewrite.
