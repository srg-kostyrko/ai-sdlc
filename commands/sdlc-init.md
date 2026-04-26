---
description: Bootstrap the ai-sdlc spec-driven workflow in the current project (creates .sdlc/ tree and appends instructions to CLAUDE.md).
argument-hint: (no arguments)
---

You are bootstrapping the ai-sdlc spec-driven workflow in the user's current project.

## Pre-flight

1. Verify the working directory looks like a project root (presence of `.git/`, `package.json`, `Cargo.toml`, `pyproject.toml`, or similar). If not, ask the user to confirm before proceeding — running this in a home directory or unintended location is a footgun.

2. Check whether `.sdlc/` already exists. If yes, the run is **idempotent**: create only what is missing, never delete or overwrite existing files inside `.sdlc/`.

## Create the marker directory

Create `.sdlc/` itself with `mkdir -p .sdlc` (idempotent). This is the marker that downstream commands check for ("Run `/ai-sdlc:sdlc-init` first" guards).

Do **not** pre-create subdirectories. The conventional layout is:

- `.sdlc/specs/<capability>/`   — bounded-context capability dirs, lazy-seeded on archive
- `.sdlc/changes/<slug>/`       — active change folders, created by `/ai-sdlc:spec-propose`
- `.sdlc/changes/archive/`      — historical, created on first archive
- `.sdlc/decisions/`            — system-wide ADRs, created when first system-wide ADR promotes
- `.sdlc/steering/`             — project-wide context, created by `/ai-sdlc:sdlc-project-init` or `/ai-sdlc:sdlc-steering`
- `.sdlc/research/`             — pre-change research notes, created by `/ai-sdlc:spec-research`

Each command `mkdir -p`s its own target right before writing, so empty placeholder dirs (which git wouldn't track anyway) never appear.

## Steering directory

Steering files are populated by:

- `/ai-sdlc:sdlc-project-init` — for greenfield projects (no source files yet); guided interview produces `product.md` and `tech.md`.
- `/ai-sdlc:sdlc-steering` — for projects with existing code; analyzes the codebase to bootstrap `product.md`, `structure.md`, `tech.md`, or syncs them when they exist.

## Update CLAUDE.md

For `CLAUDE.md` at the project root:

- If the file does not exist, create it with **only** the marked block below.
- If the file exists and contains the marker pair `<!-- ai-sdlc:start -->` … `<!-- ai-sdlc:end -->`, **replace** the content between the markers with the block below.
- If the file exists but has no marker, **append** the marked block (with a leading blank line for separation).

The marked block:

```
<!-- ai-sdlc:start -->
## Spec-driven workflow

This project uses ai-sdlc. The living specification is in `.sdlc/specs/`, organized by bounded-context capability. Active changes are in `.sdlc/changes/`; archived changes in `.sdlc/changes/archive/`.

**Project-wide context** lives in `.sdlc/steering/` (`product.md`, `structure.md`, `tech.md`). Populate via `/ai-sdlc:sdlc-project-init` (greenfield) or `/ai-sdlc:sdlc-steering` (existing code); commands read these when authoring designs and grounding answers.

**When proposing or implementing changes:**
- Start a change with `/ai-sdlc:spec-propose "<one-line seed>"` (the seed is optional; `/ai-sdlc:spec-propose` runs a short interview and derives the slug).
- Iterate with `/ai-sdlc:spec-requirements`, `/ai-sdlc:spec-design`, `/ai-sdlc:spec-tasks`.
- Generate the validation skeleton with `/ai-sdlc:spec-validate`.
- Implement one vertical slice at a time with `/ai-sdlc:spec-impl-task <task-id>`.
- Archive a complete change with `/ai-sdlc:spec-archive` (mechanical merge into the living spec).

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
.sdlc/:                created  (subdirs created lazily by downstream commands)
CLAUDE.md:             appended ai-sdlc block
Next:                  /ai-sdlc:sdlc-project-init  (greenfield, no source files yet)
                  or:  /ai-sdlc:sdlc-steering      (existing code — bootstrap from codebase)
                  or:  /ai-sdlc:spec-propose "<one-line seed>"     (start a change directly; populate steering later)
```

## Constraints

- Never delete or overwrite anything outside the `.sdlc/` tree, except the marked block in `CLAUDE.md`.
- Never modify files inside `.sdlc/changes/archive/` even on subsequent runs.
- Do not seed `.sdlc/specs/` with example capabilities; lazy seeding is intentional.
