---
description: Bootstrap or sync project-wide steering files (.sdlc/steering/) by analyzing the codebase. Use after code exists; on greenfield projects, redirect to /ai-sdlc:sdlc-project-init.
argument-hint: (no arguments)
---

You are bootstrapping or syncing the project's `.sdlc/steering/` files based on the actual codebase.

## Step 1 — Scenario detection

Check `.sdlc/steering/` and the codebase to choose a mode.

**Greenfield redirect.** If `.sdlc/steering/product.md` does not exist AND no source files are present (glob `**/*.{ts,tsx,js,jsx,py,go,rs,java,rb,cs,kt,swift,ex,erl,clj,scala,php,c,cpp,h,hpp}`):

> This repo looks empty — run `/ai-sdlc:sdlc-project-init` for guided setup.

Stop.

**Bootstrap mode.** Source files exist but core steering files are missing (`product.md`, `structure.md`, or `tech.md`).

**Sync mode.** All three core steering files exist.

## Step 2 — Bootstrap mode

Goal: produce `product.md`, `tech.md`, and `structure.md` from codebase evidence.

### Analyze (just-in-time)

- Glob for source files; note language(s) and rough file count.
- Read the project README if present.
- Read `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `composer.json`, etc., if present.
- Read other obvious config: `tsconfig.json`, `vite.config.*`, `Dockerfile`, `Makefile`, lockfiles for resolved versions.
- Grep for patterns: directory layout, common conventions, import styles.

### Extract patterns (not catalogs)

For each artifact, focus on patterns that guide future decisions, not exhaustive lists:

- **product.md** — Purpose, value, core capabilities. Synthesize from README + package metadata + obvious entry points.
- **tech.md** — Frameworks, runtime versions, key libraries (only those that influence patterns), development standards (lint config, type strictness, test framework), common commands (from `package.json` scripts or Makefile).
- **structure.md** — Organization philosophy, directory patterns with purpose and example, naming conventions, import organization, code-organization principles. Document patterns once; do not enumerate every file.

### Generate

Read each template from `${CLAUDE_PLUGIN_ROOT}/templates/steering/`. Substitute placeholder text with extracted content. Omit sections that don't apply (e.g. drop a *Database* section for a CLI tool).

Write to `.sdlc/steering/product.md`, `tech.md`, `structure.md`.

### Report

```
Bootstrap mode:
  .sdlc/steering/product.md     created
  .sdlc/steering/tech.md        created
  .sdlc/steering/structure.md   created

Source files surveyed: <N>
Patterns extracted:    <summary>
```

If hard-to-reverse decisions were detected (database choice, deployment platform, monorepo vs polyrepo, etc.), list them as ADR candidates at the end.

## Step 3 — Sync mode

Goal: detect drift between current code and existing steering; propose **additive** updates that preserve user-authored content.

### Load existing steering

Read every file in `.sdlc/steering/` (including any custom files beyond the core three).

### Analyze codebase (just-in-time)

Same survey as bootstrap mode. Look specifically for things that **changed** since steering was last written.

### Detect drift

Three categories:

1. **Steering → Code** (steering claims something the code no longer matches).
   Example: `tech.md` lists React 17, `package.json` shows React 19.
   Severity: warning. Surface for the user to confirm, then update.
2. **Code → Steering** (codebase has a pattern not yet in steering).
   Example: a new top-level directory not described in `structure.md`; a major library not in `tech.md`.
   Severity: update candidate. Propose an additive change.
3. **Custom files** (steering file beyond product/structure/tech).
   Check whether it's still relevant; if its claims contradict the code, surface as a warning.

### Propose updates

For each drift item, present:

```
[<file>] <one-line summary>
  Current steering: ...
  Codebase reality: ...
  Proposed update:  ...
```

Apply only after the user confirms (a single batch confirmation is fine).

**Update philosophy: add, don't replace.** Preserve user-authored sections. When wording must change, surface the diff.

### Report

```
Sync mode:
  Updated:    <list>
  Warnings:   <list>
  No drift:   <list>
```

## Granularity principle

> "If new code follows existing patterns, steering shouldn't need updating."

Document patterns and principles, not exhaustive lists. Steering files that turn into catalogs of dependencies or file trees rot fast.

## Constraints

- Never include keys, passwords, or secrets.
- Always add rather than replace when in doubt.
- Avoid documenting agent-specific directories (`.claude/`, `.cursor/`, `.idea/`, etc.).
- Light references to `.sdlc/specs/` and `.sdlc/steering/` are acceptable.
- Do not write a glossary file — domain terms live in per-capability `GLOSSARY.md` files, populated through change proposals.
