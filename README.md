# ai-sdlc

Spec-driven workflow for Claude Code. Living specs per bounded context, change proposals as deltas, vertical-slice tasks, ADRs as first-class.

> **Note:** This project was created for personal use. It is not intended to be supported, maintained, or extended for others. Feel free to explore and fork, but no guarantees are made regarding stability, documentation, or responsiveness to issues and pull requests.


## What it is

A Claude Code plugin that takes you from a one-line idea to a merged change through a phased pipeline:

```
/ai-sdlc:spec-propose → /ai-sdlc:spec-requirements → /ai-sdlc:spec-design → /ai-sdlc:spec-tasks → /ai-sdlc:spec-validate → /ai-sdlc:spec-impl-task → /ai-sdlc:spec-archive
```

The **living specification** accumulates in `.sdlc/specs/` per bounded-context capability. **Active changes** live in `.sdlc/changes/<slug>/` as proposal + design + tasks + validation + per-capability deltas. On archive, deltas merge mechanically into the living spec.

Phase gates are **completeness checks** — no TODOs in required sections, all slugs resolve. There is no status flag to flip; running the next phase command is the acceptance signal.

## Install

Add this repository as a Claude Code plugin (via `/plugin` marketplace install, or symlink into your plugins directory). Then in any target project:

```
/ai-sdlc:sdlc-init
```

This creates `.sdlc/` and appends a workflow section to `CLAUDE.md` so Claude discovers the convention.

## Quick start

```
/ai-sdlc:spec-propose "support TOTP for admin accounts"
# answer one question (the problem); review proposal.md
/ai-sdlc:spec-requirements          # refine narrative and deltas
/ai-sdlc:spec-design                # Approach, Goals/Non-Goals, File Structure Plan, Risks
/ai-sdlc:spec-tasks                 # decompose into vertical slices
/ai-sdlc:spec-validate              # generate validation skeleton from deltas
/ai-sdlc:spec-impl-task 1.1         # implement one slice (repeat per task)
/ai-sdlc:spec-archive               # mechanical merge into the living spec
```

If `/ai-sdlc:spec-archive` aborts on slug conflicts (another change archived first):

```
/ai-sdlc:spec-rebase                # reconcile this change's deltas against current truth
/ai-sdlc:spec-archive
```

## Concepts

- **Capability** — a bounded context (`auth`, `billing`, `notifications`, ...). Created lazily on first reference; never declared up front.
- **Living spec** — `.sdlc/specs/<capability>/spec.md` (requirements) + `GLOSSARY.md` (terms) + `CHANGELOG.md` (history) + `decisions/` (per-capability ADRs). Current truth.
- **Change** — `.sdlc/changes/<slug>/`. In-flight work bundling narrative, technical design, task breakdown, validation evidence, and one or more per-capability deltas.
- **Delta** — `changes/<slug>/specs/<capability>/delta.md`. Expresses `ADDED` / `MODIFIED` / `REMOVED` blocks for both Requirements and Terms. Merges into the living spec at archive.
- **Vertical slice** — every task satisfies ≥1 requirement (referenced by slug). No infra-only or scaffolding tasks; setup folds into the first slice that needs it.
- **ADR** — written only when a decision is (1) hard to reverse, (2) surprising without context, (3) a real trade-off. Per-capability ADRs live alongside the spec; system-wide ADRs at `.sdlc/decisions/`.
- **Steering** — `.sdlc/steering/{product,structure,tech}.md`. Project-wide context (goals, layout, stack). Populated via `/ai-sdlc:sdlc-project-init` (greenfield interview) or `/ai-sdlc:sdlc-steering` (codebase scan / drift sync). Loaded by `/ai-sdlc:spec-design` when authoring designs and by the `spec-aware` skill when grounding answers. Distinct from per-capability glossaries (bounded-context vocabulary).
- **Guideline** — reusable process knowledge (e.g. EARS criterion phrasing, verification discipline, durable-brief rules). Lives in the plugin's `guidelines/`; commands `Read` the relevant one as part of their context loading instead of inlining the rules.

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
  steering/
    product.md            # project goals + value proposition
    structure.md          # module layout + conventions
    tech.md               # stack + constraints
```

## Layout (plugin)

```
ai-sdlc/
  .claude-plugin/{plugin,marketplace}.json
  commands/               # 11 slash commands
  skills/                 # 4 skills
  templates/              # 8 change/spec templates + 3 steering templates
  guidelines/             # reusable process knowledge (criterion-phrasing, verification, durable-briefs)
```

## Commands

| Command | Purpose |
|---|---|
| `/ai-sdlc:sdlc-init` | Bootstrap `.sdlc/` and update CLAUDE.md. |
| `/ai-sdlc:sdlc-project-init [<desc>]` | Greenfield interview — populates `steering/product.md` and `tech.md`. |
| `/ai-sdlc:sdlc-steering` | Bootstrap or sync `steering/` from the codebase (existing-code projects). |
| `/ai-sdlc:spec-propose ["<seed>"]` | One-question interview (the problem); derives the slug and creates the change folder with a proposal stub plus design/tasks skeletons. Capability + first requirement are seeded by `/ai-sdlc:spec-requirements`. |
| `/ai-sdlc:spec-requirements [<slug>]` | Refine proposal narrative + deltas. |
| `/ai-sdlc:spec-design [<slug>]` | Author Approach, Goals/Non-Goals, File Structure Plan, Risks; optionally draft ADRs. |
| `/ai-sdlc:spec-tasks [<slug>]` | Decompose into vertical slices. |
| `/ai-sdlc:spec-validate [<slug>]` | Generate / refresh validation skeleton from deltas. |
| `/ai-sdlc:spec-impl-task <id> [<slug>]` | Implement one slice; auto-tick validation rows whose tests pass; auto-tick the task box on success. |
| `/ai-sdlc:spec-archive [<slug>]` | Mechanical merge of deltas into living spec; promote ADRs; move folder to archive. |
| `/ai-sdlc:spec-rebase [<slug>]` | Reconcile deltas with current living spec after a conflict. |

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
- **No override flag.** `/ai-sdlc:spec-archive` refuses on incomplete validation, missing approvals, or slug conflicts. Fix the cause; do not bypass.
- **Auto-tick on impl success.** `/ai-sdlc:spec-impl-task` ticks validation rows whose referenced tests pass and ticks the task box if implementation completed without errors. Validation rows — not task boxes — are the contract for "requirements satisfied"; the checkbox is bookkeeping. If review reveals issues, re-run `/ai-sdlc:spec-impl-task <id>` or ask Claude to untick and revise.

## Conceptual influences

- **OpenSpec** — living-spec-vs-change-proposal model; ADDED/MODIFIED/REMOVED delta grammar.
- **cc-sdd** — phased pipeline; structured task metadata (`_Capabilities:_` / `_Requirements:_` / `_Depends:_`).
- **skills** — ADR criteria (hard to reverse / surprising / real trade-off); structured Nygard ADR format; bounded-context vocabulary discipline.
