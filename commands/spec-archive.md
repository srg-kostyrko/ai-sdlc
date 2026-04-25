---
description: Mechanically merge an active change's deltas into the living spec, promote ADRs, and move the folder to the archive. Refuses on incomplete validation or slug conflicts.
argument-hint: [<slug>]
---

You are archiving a complete change. This is mostly mechanical — do not paraphrase or "improve" delta content during the merge.

## Step 1 — Resolve the change

User input: $ARGUMENTS

- If `$ARGUMENTS` contains a slug, use `.sdlc/changes/<slug>/`. If that folder doesn't exist, list active changes and ask which one.
- If `$ARGUMENTS` is empty:
  - If exactly one folder exists under `.sdlc/changes/` (excluding `archive/`), use it.
  - Otherwise list the active changes and ask which one.

## Step 2 — Gates

Refuse unless ALL hold. Print specific failures and stop on miss. There is **no override flag**.

1. `.sdlc/changes/<slug>/tasks.md` exists, has ≥1 task, and **every task is ticked** (`- [x]`).
2. `.sdlc/changes/<slug>/validation.md` exists.
3. Every row in `validation.md` is ticked (`- [x]`).
4. Every row in `validation.md` has a non-empty `_Evidence:_` line.
5. Every row whose `_Evidence:_` starts with `manual` has an `_Approved:_` line below it with author + date.
6. No `<!-- TODO -->` markers anywhere in `proposal.md`, `design.md`, `tasks.md`, `validation.md`, or any `delta.md`.
7. `.sdlc/changes/<slug>/findings.md` exists with a `## Verdict` block.
8. `## Verdict` shows `Result: GO`.
9. The verdict's `Head ref:` value equals the current `git rev-parse HEAD`. (If commits or uncommitted changes have landed since the last review, re-run `/spec-review` to refresh the verdict.)

## Step 3 — Slug-presence sanity check (per delta)

For each `.sdlc/changes/<slug>/specs/<capability>/delta.md`:

Determine the target living files: `.sdlc/specs/<capability>/spec.md` and `.sdlc/specs/<capability>/GLOSSARY.md`. They may not yet exist (lazy seeding) — handle that branch in step 4.

If the living capability already exists, verify:

- For each ADDED requirement slug: must NOT exist in living `spec.md`.
- For each MODIFIED requirement slug: MUST exist in living `spec.md`.
- For each REMOVED requirement slug: MUST exist in living `spec.md`.
- Same three rules for ADDED/MODIFIED/REMOVED Terms against living `GLOSSARY.md`.

If any check fails, **abort** with a precise report and suggest `/spec-rebase`:

```
/spec-archive aborted. Slug conflicts detected:
  specs/auth/delta.md: req-totp-enrollment marked ADDED but already exists in living spec
  specs/auth/delta.md: req-old-totp marked MODIFIED but does not exist in living spec
Run /spec-rebase to reconcile.
```

Do not partially apply. All-or-nothing.

## Step 4 — Apply the merge

For each `delta.md`, in any order (capabilities are independent):

### Lazy seed if needed
If `.sdlc/specs/<capability>/` does not exist, create it with:
- `spec.md` from `${CLAUDE_PLUGIN_ROOT}/templates/spec.md` (substitute `{{CAPABILITY}}`).
- `GLOSSARY.md` from `${CLAUDE_PLUGIN_ROOT}/templates/glossary.md` (substitute `{{CAPABILITY}}`).
- An empty `CHANGELOG.md` containing only `# <capability> — Changelog\n`.
- Empty `decisions/` directory.

### Apply Requirements deltas to `spec.md`
- For each block under `## ADDED Requirements`: append the requirement (verbatim, header + body) to the `## Requirements` section in living `spec.md`. Maintain order: append at the end of `## Requirements`.
- For each block under `## MODIFIED Requirements`: locate the existing requirement with the same slug in living `spec.md` and **replace its block in place** (from its `### Requirement:` heading down to the next `### Requirement:` heading or end of section). Use the new content verbatim.
- For each block under `## REMOVED Requirements`: locate by slug and delete the block.

### Apply Terms deltas to `GLOSSARY.md`
Same three operations against the `## Terms` section, keyed by term slug.

### Append the CHANGELOG entry
Append one line to `.sdlc/specs/<capability>/CHANGELOG.md`:

```
<YYYY-MM-DD> <slug>: ADDED <list-of-added-req-slugs>; MODIFIED <list>; REMOVED <list>
```

(Omit segments with empty lists. Use today's date in YYYY-MM-DD.)

## Step 5 — Promote ADRs

For each `.sdlc/changes/<slug>/decisions/draft-*.md`:

1. Read the draft. Parse its `Affects:` header.
2. Choose the target directory:
   - If `Affects:` says `system-wide`: `.sdlc/decisions/`.
   - Otherwise (e.g. `Affects: specs/auth — req-...`): the named capability's `.sdlc/specs/<capability>/decisions/`. Create the directory if missing.
3. Determine the next sequence number: scan target dir for files matching `^[0-9]{4}-.*\.md$`, take the max numeric prefix, add 1. Pad to 4 digits. (If empty, start at `0001`.)
4. New filename: `<NNNN>-<slug-from-draft-filename>.md` (drop the `draft-` prefix).
5. In the file body, replace the title line `# ADR-{{NUMBER}}: <title>` with `# ADR-<NNNN>: <title>` and set `Status: accepted`.
6. Move the file to its target path.

If two parallel changes both have draft ADRs, sequence numbers are assigned in archive order — there's no collision because we always re-read the target directory at promotion time.

## Step 6 — Archive the change folder

1. Compute archive folder name: `<YYYY-MM-DD>-<slug>` (today's date).
2. `mkdir .sdlc/changes/archive/<YYYY-MM-DD>-<slug>/`.
3. **Copy** these files into the archive folder:
   - `proposal.md`
   - `design.md`
   - `validation.md`
4. **Do not** copy `tasks.md` or anything under `specs/` (the deltas) — they're now redundant with the merged living spec, and `tasks.md` is operational, not historical.
5. Delete the original `.sdlc/changes/<slug>/` (with its `tasks.md`, `specs/`, `decisions/` — note ADRs were already moved out).

## Step 7 — Report

Print a structured summary:

```
Archived <slug>:
  Living spec changes:
    specs/auth/spec.md       +<N> requirements (req-...), ~<N> modified, -<N> removed
    specs/auth/GLOSSARY.md   +<N> terms, ~<N>, -<N>
    specs/auth/CHANGELOG.md  appended entry
  ADRs promoted:
    specs/auth/decisions/0014-totp-kms-key.md  (was draft-totp-kms-key.md)
  Archive:
    changes/archive/2026-04-25-<slug>/  (proposal, design, validation)
  Removed:
    changes/<slug>/

Run `git status` and commit when ready. Suggested message:
  spec(<slug>): archive change — <one-line proposal Why>
```

## Constraints

- Mechanical merge only: never paraphrase or restructure delta content during the merge. The user's deltas are the source of truth.
- All-or-nothing: if any sanity check fails, abort with no partial changes on disk.
- Do not run `git commit` automatically. Leave the diff for the user.
- Never modify archived folders (`changes/archive/*`) — those are immutable history.
