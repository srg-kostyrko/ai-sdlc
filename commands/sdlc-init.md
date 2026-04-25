---
description: Bootstrap the ai-sdlc spec-driven workflow in the current project (creates .sdlc/ tree and appends instructions to CLAUDE.md / AGENTS.md).
argument-hint: (no arguments)
---

You are bootstrapping the ai-sdlc spec-driven workflow in the user's current project.

## Pre-flight

1. Verify the working directory looks like a project root (presence of `.git/`, `package.json`, `Cargo.toml`, `pyproject.toml`, or similar). If not, ask the user to confirm before proceeding — running this in a home directory or unintended location is a footgun.

2. Check whether `.sdlc/` already exists. If yes, the run is **idempotent**: create only what is missing, never delete or overwrite existing files inside `.sdlc/`.

## Create the tree

Create these directories (use `mkdir -p`; idempotent):

- `.sdlc/specs/`             — empty; bounded-context capabilities are created lazily on first reference
- `.sdlc/changes/`           — active change proposals
- `.sdlc/changes/archive/`   — historical, immutable
- `.sdlc/decisions/`         — system-wide ADRs only (per-capability ADRs live under `specs/{capability}/decisions/`)

## Update CLAUDE.md and AGENTS.md

For each of `CLAUDE.md` and `AGENTS.md` at the project root:

- If the file does not exist, create it with **only** the marked block below.
- If the file exists and contains the marker pair `<!-- ai-sdlc:start -->` … `<!-- ai-sdlc:end -->`, **replace** the content between the markers with the block below.
- If the file exists but has no marker, **append** the marked block (with a leading blank line for separation).

The marked block (identical in both files):

```
<!-- ai-sdlc:start -->
## Spec-driven workflow

This project uses ai-sdlc. The living specification is in `.sdlc/specs/`, organized by bounded-context capability. Active changes are in `.sdlc/changes/`; archived changes in `.sdlc/changes/archive/`.

**When proposing or implementing changes:**
- Start a change with `/spec-propose <slug> "<one-line description>"`.
- Iterate with `/spec-requirements`, `/spec-design`, `/spec-tasks`.
- Generate the validation skeleton with `/spec-validate`.
- Implement one vertical slice at a time with `/spec-impl-task <task-id>`.
- Archive a complete change with `/spec-archive` (mechanical merge into the living spec).

**Phase gates are completeness checks, not flags.** Each command refuses if its prior artifact has unresolved `<!-- TODO -->` markers, missing required sections, or dangling slug references. To go backward, re-run the prior phase command.

**Vertical slices only.** Every task in `tasks.md` must satisfy at least one requirement (referenced by slug). No "infra" or "scaffolding" tasks; setup work folds into the first slice that needs it.

**Do not:**
- Edit `.sdlc/specs/{capability}/spec.md` or `GLOSSARY.md` by hand — changes flow through deltas merged at archive time.
- Modify anything under `.sdlc/changes/archive/` — archived changes are immutable history.
<!-- ai-sdlc:end -->
```

## Report

Print a short summary of what was created vs already present, e.g.:

```
.sdlc/ tree:    created  (specs/, changes/, changes/archive/, decisions/)
CLAUDE.md:      appended ai-sdlc block
AGENTS.md:      created with ai-sdlc block
Next step:      /spec-propose <slug> "<description>"
```

## Constraints

- Never delete or overwrite anything outside the `.sdlc/` tree, except the marked block in `CLAUDE.md` / `AGENTS.md`.
- Never modify files inside `.sdlc/changes/archive/` even on subsequent runs.
- Do not seed `.sdlc/specs/` with example capabilities; lazy seeding is intentional.
