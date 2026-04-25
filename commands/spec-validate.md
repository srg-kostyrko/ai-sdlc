---
description: Generate or refresh validation.md from the change's deltas. Drafts in memory, runs a review gate, writes only when clean. Preserves existing evidence and approvals.
argument-hint: [<slug>]
---

You are generating the validation skeleton (or refreshing it) for an active change.

## Step 1 — Resolve the change

User input: $ARGUMENTS

- If `$ARGUMENTS` contains a slug, use `.sdlc/changes/<slug>/`. If that folder doesn't exist, list active changes and ask which one.
- If `$ARGUMENTS` is empty:
  - If exactly one folder exists under `.sdlc/changes/` (excluding `archive/`), use it.
  - Otherwise list the active changes and ask which one.

## Step 2 — Gate

The only gate: at least one `.sdlc/changes/<slug>/specs/*/delta.md` must exist with a non-empty ADDED, MODIFIED, or REMOVED Requirements block. No prior-artifact gate; validation can be generated as soon as deltas exist, and re-run any time.

If the gate fails, print:

```
/spec-validate refused. Reason:
- No non-empty deltas found under .sdlc/changes/<slug>/specs/
Run /spec-requirements to author requirements first.
```

## Step 3 — Read context

Read into main context:

- All deltas under `.sdlc/changes/<slug>/specs/` (collect every ADDED/MODIFIED scenario, every criterion, every REMOVED requirement slug)
- Existing `.sdlc/changes/<slug>/validation.md` if present — used to preserve filled evidence

## Step 4 — Build the skeleton in memory

Read `${CLAUDE_PLUGIN_ROOT}/templates/validation.md` for the canonical structure.

Generate the new skeleton, in memory, with one section per capability and subsections per requirement:

For every **ADDED** requirement:
- Subsection: `### <req-slug> (ADDED)`
- One row per `#### Scenario:` block: `- [ ] Scenario: <name>` with `_Evidence:_` empty.
- One row per bullet under `#### Criteria`: `- [ ] Criterion: <text>` with `_Evidence:_` empty.

For every **MODIFIED** requirement:
- Subsection: `### <req-slug> (MODIFIED)`
- Same row generation as ADDED, but for the *modified* scenarios/criteria as they appear in the delta.

For every **REMOVED** requirement:
- Subsection: `### <req-slug> (REMOVED)`
- One negative-evidence row: `- [ ] Smoke: <prior behavior> must NO LONGER hold` with `_Evidence:_` empty.

## Step 5 — Merge with existing validation.md

If `validation.md` exists, merge into the new skeleton (in memory). Do not overwrite.

Key each row by `(req-slug, ADDED|MODIFIED|REMOVED, scenario-or-criterion-text)`.

Three cases:

- **Match found, existing row has `[x]` or non-empty `_Evidence:_`** → preserve the existing row verbatim (including `_Approved:_` if present). New rows generated for the same key are discarded.
- **No match in existing file** → keep the new empty row.
- **Existing row has no match in new skeleton** → the underlying delta entry was removed or its text changed. **Hold for review**; do not silently drop.

## Step 6 — Review gate

Run mechanical checks. (Judgment checks are minimal here since the skeleton is mechanically derived from deltas.)

### Mechanical checks (must all pass)

1. The merged draft has ≥1 row.
2. Every ADDED/MODIFIED scenario in the deltas has a corresponding row in the draft.
3. Every bullet under `#### Criteria` (for ADDED/MODIFIED requirements) has a corresponding row.
4. Every REMOVED requirement has a negative-evidence row.
5. Every preserved evidence row's `(req-slug, type, text)` key still maps to a delta entry. (Stale rows are flagged in Step 5 and must be resolved before the gate passes.)
6. Every preserved row whose `_Evidence:_` starts with `manual` has an `_Approved:_` line.

### Judgment check (single)

1. Stale rows (existing rows with no match in the new skeleton) are surfaced to the user with their `(req-slug, scenario/criterion text)` key. The user explicitly chooses: drop, restore (if their delta change was unintentional), or keep as a custom row (rare).

### Resolution

For each stale row, ask the user one question and act on the answer. There is no automatic repair loop here — stale rows are user choices, not draft defects.

## Step 7 — Finalize

Once the review gate passes (and stale rows are resolved), write the merged draft to `.sdlc/changes/<slug>/validation.md`.

Report:

```
validation.md:               written (<N> rows)
  Added scenarios:           <count>
  Added criteria:            <count>
  Modified scenarios:        <count>
  Modified criteria:         <count>
  Removed (negative):        <count>
  Preserved (filled):        <count>   <- evidence + approvals carried over
  Stale rows resolved:       <count>   (dropped: <N>, restored: <N>, kept-custom: <N>)

Next:
  - Tasks unticked: run /spec-impl-task <id> for each. Auto-ticks rows whose tests pass.
  - Tasks ticked, rows empty: fill _Evidence:_ (manual rows need _Approved:_).
  - All rows green: ready for /spec-archive.
```

If stale rows remain unresolved (user deferred decisions), write **nothing** and report:

```
Validation refresh halted.
Unresolved stale rows: <N>. Decide drop/restore/keep for each and re-run.
```

## Constraints

- Never modify deltas, design, tasks, or proposal — `/spec-validate` only rewrites `validation.md`.
- Never invent test references or approvals. Empty `_Evidence:_` is correct for new rows.
- Preserve user-authored evidence lines verbatim during refresh, including formatting and any `_Approved:_` line.
- Do not silently delete rows. Stale rows are surfaced and decided explicitly.

## Error scenarios

- **No deltas yet (gate fail).** Suggest `/spec-requirements` to author requirements.
- **Delta has no scenarios or criteria.** A requirement is declared ADDED/MODIFIED but has no `#### Scenario:` and no `#### Criteria` content → no rows can be generated. Ask user to flesh out the delta via `/spec-requirements`.
- **Existing row's underlying scenario text changed.** Existing row preserved against scenario "Admin enrolls"; delta now has scenario "Admin enrolls successfully" → flagged as stale. User decides: rename evidence to new key, or drop and re-author.
- **Manual row missing approval.** Preserved row has `_Evidence:_ manual ...` but no `_Approved:_` line → mechanical check fails. Ask user to add approval or convert to test-backed evidence.
- **Skeleton template missing.** `${CLAUDE_PLUGIN_ROOT}/templates/validation.md` not readable → fall back to inline structure (defined in this command), warn the user that templates may be misconfigured.
