# ai-sdlc

Spec-driven workflow for Claude Code. Living specs per bounded context, change proposals as deltas, vertical-slice tasks, ADRs as first-class.

> **Note:** This project was created for personal use. It is not intended to be supported, maintained, or extended for others. Feel free to explore and fork, but no guarantees are made regarding stability, documentation, or responsiveness to issues and pull requests.


## What it is

A Claude Code plugin that takes you from a one-line idea to a merged change through a phased pipeline:

```
/spec-propose → /spec-requirements → /spec-design → /spec-tasks → /spec-validate → /spec-impl-task → /spec-archive
```

The **living specification** accumulates in `.sdlc/specs/` per bounded-context capability. **Active changes** live in `.sdlc/changes/<slug>/` as proposal + design + tasks + validation + per-capability deltas. On archive, deltas merge mechanically into the living spec.

Phase gates are **completeness checks** — no TODOs in required sections, all slugs resolve. There is no status flag to flip; running the next phase command is the acceptance signal.

## Install

Add this repository as a Claude Code plugin (via `/plugin` marketplace install, or symlink into your plugins directory). Then in any target project:

```
/sdlc-init
```

This creates `.sdlc/` and appends a workflow section to `CLAUDE.md` and `AGENTS.md` so Claude (and other agents) discover the convention.

## Quick start

```
/spec-propose add-2fa "support TOTP for admin accounts"
# review proposal.md and the initial delta
/spec-requirements          # refine narrative and deltas
/spec-design                # Approach, Goals/Non-Goals, File Structure Plan, Risks
/spec-tasks                 # decompose into vertical slices
/spec-validate              # generate validation skeleton from deltas
/spec-impl-task 1.1         # implement one slice (repeat per task)
/spec-archive               # mechanical merge into the living spec
```

If `/spec-archive` aborts on slug conflicts (another change archived first):

```
/spec-rebase                # reconcile this change's deltas against current truth
/spec-archive
```

## Concepts

- **Capability** — a bounded context (`auth`, `billing`, `notifications`, ...). Created lazily on first reference; never declared up front.
- **Living spec** — `.sdlc/specs/<capability>/spec.md` (requirements) + `GLOSSARY.md` (terms) + `CHANGELOG.md` (history) + `decisions/` (per-capability ADRs). Current truth.
- **Change** — `.sdlc/changes/<slug>/`. In-flight work bundling narrative, technical design, task breakdown, validation evidence, and one or more per-capability deltas.
- **Delta** — `changes/<slug>/specs/<capability>/delta.md`. Expresses `ADDED` / `MODIFIED` / `REMOVED` blocks for both Requirements and Terms. Merges into the living spec at archive.
- **Vertical slice** — every task satisfies ≥1 requirement (referenced by slug). No infra-only or scaffolding tasks; setup folds into the first slice that needs it.
- **ADR** — written only when a decision is (1) hard to reverse, (2) surprising without context, (3) a real trade-off. Per-capability ADRs live alongside the spec; system-wide ADRs at `.sdlc/decisions/`.

## Layout (target project)

```
.sdlc/
  specs/
    <capability>/
      spec.md             # ## Purpose + ## Requirements
      GLOSSARY.md         # ## Terms
      CHANGELOG.md        # one line per archived change
      decisions/
        NNNN-slug.md      # per-capability ADRs
  changes/
    <slug>/               # active proposal
      proposal.md
      design.md
      tasks.md
      validation.md
      specs/<capability>/delta.md
      decisions/draft-*.md
    archive/
      YYYY-MM-DD-<slug>/  # archived (proposal + design + validation only)
  decisions/
    NNNN-slug.md          # system-wide ADRs
```

## Layout (plugin)

```
ai-sdlc/
  .claude-plugin/{plugin,marketplace}.json
  commands/               # 9 slash commands
  skills/                 # 4 skills
  templates/              # 8 markdown templates
```

## Commands

| Command | Purpose |
|---|---|
| `/sdlc-init` | Bootstrap `.sdlc/` and update CLAUDE.md / AGENTS.md. |
| `/spec-propose <slug> "<desc>"` | Create change folder with first-draft artifacts. |
| `/spec-requirements [<slug>]` | Refine proposal narrative + deltas. |
| `/spec-design [<slug>]` | Author Approach, Goals/Non-Goals, File Structure Plan, Risks; optionally draft ADRs. |
| `/spec-tasks [<slug>]` | Decompose into vertical slices. |
| `/spec-validate [<slug>]` | Generate / refresh validation skeleton from deltas. |
| `/spec-impl-task <id> [<slug>]` | Implement one slice; auto-tick validation rows whose tests pass; auto-tick the task box on success. |
| `/spec-archive [<slug>]` | Mechanical merge of deltas into living spec; promote ADRs; move folder to archive. |
| `/spec-rebase [<slug>]` | Reconcile deltas with current living spec after a conflict. |

When a single active change exists, `<slug>` is optional on phase commands.

## Skills

- **spec-aware** — passive context loader. Activates when you touch a bounded context; silently reads the relevant capability spec and glossary so answers are grounded in current truth.
- **spec-lookup** — find a requirement, term, scenario, or ADR by keyword across living spec + in-flight deltas.
- **spec-glossary-suggest** — flag domain noun phrases that should be defined in the capability glossary.
- **spec-scenario-rewrite** — convert prose acceptance criteria into WHEN/THEN scenarios.

## Design notes

- **Lazy capability seeding.** No up-front list of bounded contexts. The first proposal targeting a new capability creates it on archive.
- **Slug-based identity.** Requirements use `{#req-slug}`; terms use `{#term-slug}`. Slugs never change; titles may drift.
- **Mechanical archive.** Slug-presence sanity check, then ADDED appends, MODIFIED replaces, REMOVED deletes. No paraphrasing during merge.
- **No override flag.** `/spec-archive` refuses on incomplete validation, missing approvals, or slug conflicts. Fix the cause; do not bypass.
- **Auto-tick on impl success.** `/spec-impl-task` ticks validation rows whose referenced tests pass and ticks the task box if implementation completed without errors. Validation rows — not task boxes — are the contract for "requirements satisfied"; the checkbox is bookkeeping. If review reveals issues, re-run `/spec-impl-task <id>` or ask Claude to untick and revise.

## Conceptual influences

- **OpenSpec** — living-spec-vs-change-proposal model; ADDED/MODIFIED/REMOVED delta grammar.
- **cc-sdd** — phased pipeline; `_Boundary:_` / `_Depends:_` / parallel-marker task metadata.
- **skills** — ADR criteria (hard to reverse / surprising / real trade-off); structured Nygard ADR format; bounded-context vocabulary discipline.
