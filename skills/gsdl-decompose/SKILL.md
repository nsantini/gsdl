---
name: gsdl-decompose
description: Break SPEC.md down into implementable slices, write tasks.md, and create one tracker sub-item per slice when a tracker is configured. Use when the user wants to decompose a spec into tasks, break a plan into sub-issues, or create a task list from SPEC.md. This is Step 4 (Decompose into Slices) of the GSDL pipeline. Slicing runs on the LARGE tier; the tracker writes are delegated to the SMALL tier via gsdl-tracker-sync.
tier: large
disable-model-invocation: true
---

# Decompose into Slices

Breaks `SPEC.md` into implementable slices and — when a tracker is configured — creates one
**sub-item per slice** under the parent work item. This is the **Step 4 checkpoint**:
`Pick up source → Brainstorm → SPEC.md → Slices → Review → Branch → Execute → Verify → PR + bot → Close loop`

`tasks.md` is always written: it is what `gsdl-execute` reads from during implementation.

| Tracker config | Checkpoint artifact |
|---|---|
| `linear` / `jira` / `github` / `notion` | Sub-items in the tracker, each carrying its sub-task detail, **plus** `tasks.md` |
| `none` | `tasks.md` alone — the slices are reviewed there |

## Tier

**LARGE for the slicing; SMALL for the tracker writes.**

Slice boundaries, dependency ordering, and what counts as independently shippable are the
highest-leverage judgment calls in the pipeline — each slice becomes one MEDIUM subagent's entire
brief in Step 6, and a bad cut is expensive to unwind once sub-items exist in a tracker other people
can see. That thinking is LARGE.

Creating the items afterwards is not: it's resolving label IDs and POSTing mutations. Hand that to a
**SMALL subagent running `gsdl-tracker-sync`** (Operation A) once the slices are settled.

The `gsdl` orchestrator runs this step's slicing on LARGE either way — inline when the session is
already on a LARGE model, otherwise in a LARGE subagent using its Question Relay protocol. **Never
run the slicing on a smaller tier**, and never ask the user to switch models themselves.

## Prerequisites

1. **Project exists**: `.planning/[project-name]/`
2. **`SPEC.md` exists**: `.planning/[project-name]/SPEC.md`
3. **Provider config resolved**: `.planning/gsdl.config.md` — the `## Tracker` section decides
   whether sub-items are created at all
4. **Parent item known** *(only when a tracker is configured)*: the item ID recorded in `seed.md`'s
   `## Source` section

**If a tracker is configured but `seed.md` records no item** (the project started from a document or
a plain idea), ask the user once:

```
This project has no [tracker] item. Do you want me to:
  a) create a parent item first, and hang the slices off it
  b) create the slices as standalone items with no parent
  c) skip the tracker for this project and keep the slices in tasks.md only
```

Record the answer in the project's config (`.planning/[project-name]/gsdl.config.md`) so it is never
asked twice. Do not create orphaned sub-items by default.

## Output

- **Local**: `.planning/[project-name]/tasks.md` (always)
- **Tracker**: one sub-item per parent slice, parented to the item from `seed.md` (when configured)

## Two-Phase Process

### Phase 1: Generate Parent Slices

1. **Read `SPEC.md` from disk** — never rely on an in-context version; the user may have edited it.
2. **Analyze** the functional requirements, user stories, and verification plan.
3. **Size the decomposition** — decide how many parent slices the work actually needs (see "How Many
   Slices?" below). There is no fixed target; a one-file change is one slice.
4. **Create that many parent slices** — high-level, independently implementable chunks of work.
5. **Present the slices to the user** (without sub-tasks yet):

```markdown
## Slices

- [ ] 1.0 Parent Slice Title
- [ ] 2.0 Parent Slice Title
...
```

6. **State the sizing rationale in one line** — e.g. "3 slices: single service, no schema or API
   changes, one integration point."
7. **Pause for confirmation**: "I've generated the high-level slices from SPEC.md. Review them above
   — feel free to edit SPEC.md or suggest changes before we continue. Respond with 'Go' to generate
   sub-tasks and create the sub-items."

### Phase 2: Generate Sub-Tasks and Create Sub-Items

1. **Wait for "Go"**.
2. **Re-read `SPEC.md` from disk** to pick up edits made during Phase 1's review window.
3. **Break down each slice** into concrete, actionable sub-tasks.
4. **Identify relevant files** likely to be created or modified per slice.
5. **Create one tracker sub-item per parent slice** — delegate the API work to SMALL (below). Skip
   entirely with `Tracker → provider: none`.
6. **Write `tasks.md`**, embedding each sub-item's identifier next to its parent slice for
   traceability.
7. **Show the checkpoint**: the created sub-item links and the local file path, then wait for the
   Step 5 human review before Step 6 begins.

## Creating Sub-Items in the Tracker

**You decide what the sub-items say; `gsdl-tracker-sync` creates them.** Once the user has confirmed
the slices, spawn a **SMALL subagent** running `gsdl-tracker-sync` (Operation A) and pass it:

- `PARENT_ITEM_ID` — the item from `seed.md`, or `none`
- `PROJECT_ID` — the parent item's team/project/repo, from the config
- `SLICES` — the ordered list of `{ title, description }`, where `description` is the slice's
  sub-tasks as a checklist plus its "Relevant Files"

That skill owns the mechanics and the invariants every sub-item must satisfy — the configured
`labels`, parenting, and the provider-specific field mapping. It returns each slice's `identifier`
and `url`, in order — use those to tag `tasks.md`.

### If the tracker is unreachable

`gsdl-tracker-sync` returns a BLOCKER with the payload formatted for manual entry. Show the user the
full slice breakdown, ask them to create the sub-items manually with the same labels, and have them
paste back the resulting IDs for `tasks.md`. Do not proceed to Step 5 with untagged slices — or,
with the user's explicit agreement, switch this project to `Tracker → provider: none` and continue
with `tasks.md` as the record.

## `tasks.md` Format

```markdown
## Relevant Files

- `src/path/to/file1.ts` - Brief description (e.g., main component for this feature).
- `src/path/to/file1.test.ts` - Unit tests for `file1.ts`.

### Notes

- Implementation file paths are relative to the workspace root. Planning files live under `.planning/[project-name]/`.
- Unit tests are typically placed alongside the code files they test.
- Run the project's test command (see SPEC.md's Verification Plan) to check individual files.

## Tasks

- [ ] 1.0 [ENG-124] Parent Slice Title
  - [ ] 1.1 Sub-task description
  - [ ] 1.2 Sub-task description
- [ ] 2.0 [ENG-125] Parent Slice Title
  - [ ] 2.1 Sub-task description
```

The bracketed tag next to each parent slice is its tracker sub-item identifier — `gsdl-execute` and
`gsdl-close-loop` use it to post updates back to the right item. Tag format follows the tracker:
`[ENG-124]` (Linear), `[PROJ-45]` (Jira), `[#128]` (GitHub). **With no tracker, omit the tag
entirely** — do not invent placeholder IDs.

## How Many Slices?

**Slice count follows the work, not a template.** Padding a small change into 5 slices creates
busywork: more items to review, more subagent handoffs in Step 6, more tracker noise for something
one agent could ship in a single pass. Under-slicing is the opposite failure — a slice too big for
one MEDIUM subagent's brief gets a vague cut and a messy diff.

Derive the count from what `SPEC.md` actually contains:

| Signals in SPEC.md | Typical slices |
|---|---|
| One file/module, no new dependencies, single obvious verification step | **1** |
| A few files in one area, one behaviour change, tests alongside | **2-3** |
| Multiple modules or layers (e.g. API + UI), a migration, or several distinct user stories | **4-6** |
| Cross-service, new subsystem, schema + backfill + rollout, many independent stories | **7+** |

Weigh these, in rough order of importance:

1. **Number of independently shippable/reviewable units** — the primary driver; each slice should be
   a diff a human would want to review on its own.
2. **Distinct functional requirements and user stories** in `SPEC.md` — several tightly coupled
   requirements can share one slice; unrelated ones should not.
3. **Layer/service boundaries crossed** — each boundary usually implies at least one slice.
4. **Sequencing constraints** — work that must land before other work can start is its own slice.
5. **Fit for one subagent** — a slice should be a brief a single MEDIUM subagent can complete and
   verify in one pass. Too large to hold at once → split; too trivial to be worth its own item →
   merge into a neighbour.

Rules of thumb:

- **Minimum is 1.** If the whole spec is one coherent, independently shippable change, create one
  slice and say so — do not manufacture "Set up structure" / "Add tests" / "Update docs" slices just
  to reach a count. Scaffolding-only and test-only slices are almost always a symptom of
  over-slicing; fold them into the slice whose behaviour they support.
- **Above ~8 slices**, consider whether `SPEC.md` is really one story — flag it and suggest splitting
  the parent item instead.
- **State your reasoning** — always tell the user why you chose the count you did, so they can
  correct it during the Phase 1 pause.

## Slice Breakdown Guidelines

### Parent Slices
- As many high-level slices as the work needs — commonly 1-3 for small changes, 4-6 for
  feature-sized work
- Numbered `1.0`, `2.0`, `3.0`, ...
- Each should be independently reviewable and, ideally, independently shippable

### Sub-Tasks
- Nested numbering `1.1`, `1.2`, ... under parent `1.0`
- Concrete and implementable, logically following from the parent slice
- Not all parents need sub-tasks (e.g., simple configuration slices)

### Relevant Files
- List every file expected to be created/modified, implementation + tests
- Full paths relative to workspace root
- One-line purpose per file

## Interaction Model

1. Generate parent slices → show to user → **wait for "Go"**
2. User confirms → generate sub-tasks, create tracker sub-items, write `tasks.md`

Do not create tracker sub-items until the user has explicitly confirmed the slice breakdown — item
creation is harder to cleanly undo than editing a markdown file, and other people can see it.

## Rules

1. **Never skip Phase 1's pause** — sub-item creation is real, user-visible activity in a shared
   tracker; get sign-off on the shape first
2. **Never pad the slice count** — match it to the work; one slice is a valid answer, and every slice
   must justify its own item
3. **Always tag each parent slice with its tracker identifier** in `tasks.md` once created, and never
   invent one when there is no tracker
4. **Prefer updating over duplicating** — if `tasks.md` and matching sub-items already exist for this
   project, ask whether to amend them rather than creating a parallel set
5. **Never call a tracker API yourself** — the writes go to a SMALL subagent running
   `gsdl-tracker-sync`, which owns the provider-specific invariants
6. **`Tracker → provider: none` is a complete path**, not a degraded one — `tasks.md` is the
   checkpoint, and Step 5's review happens against it
