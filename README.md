# GSDL — Get Shit Done Light

A collection of Cursor Agent Skills that guide you through a full project development pipeline, from raw idea to working implementation.

## Overview

GSDL provides five skills that can be used individually or chained together as a complete end-to-end workflow:

```
Setup Project → Create PRD → Create Plan → Execute Plan
```

An orchestrator skill (`gsdl`) coordinates all four phases automatically, with human checkpoints between each step.

---

## Skills

### `gsdl` — Full Pipeline Orchestrator

Runs the entire workflow end-to-end. Detects which phase a project is at and resumes from there. Phases 0–2 run inline (interactively in the current context). Phase 3 spawns one subagent per parent task to keep context fresh.

**Trigger phrases:** "let's GSD", "build this from scratch", "get shit done", or "run the full workflow"

**Pipeline:**
```
Phase 0: Setup       → creates project folder and seed file
Phase 1: PRD         → interactive Q&A to produce a requirements doc
Phase 2: Task List   → generates parent tasks → waits for "Go" → generates sub-tasks
Phase 3: Implement   → one subagent per parent task, with checkpoint between each
```

A checkpoint is shown between every phase and between every parent task during implementation. The pipeline never auto-advances past a checkpoint without your confirmation.

At each checkpoint, you can **edit the artifact directly** (the PRD, task list, etc.) before confirming. The next phase always re-reads from disk, so your edits are automatically picked up.

---

### `gsdl-setup-project` — Project Setup

Creates a standardized folder structure for a new project.

**Trigger phrases:** "start a new project", "set up a project", "initialize a new idea"

**Creates:**
```
.planning/
└── project-name/
    ├── seed.md        ← initial idea capture
    └── tasks/         ← PRDs and task lists go here
```

The `seed.md` file is intentionally informal — it captures your raw thinking (problem, rough feature ideas, questions, references) before anything gets formalized.

---

### `gsdl-create-prd` — Create PRD

Generates a structured Product Requirements Document from your seed file or a verbal description. Asks clarifying questions before writing anything.

**Trigger phrases:** "create a PRD", "document requirements", "turn this idea into a spec"

**Output:** `.planning/[project-name]/tasks/prd-[feature-name].md`

The PRD covers: goals, user stories, functional requirements, non-goals, design/technical considerations, success metrics, and open questions. Written to be clear enough for a junior developer to implement from.

---

### `gsdl-create-plan` — Generate Task List

Breaks down a PRD into a hierarchical, checkbox-tracked task list. Uses a two-phase process: shows parent tasks first, waits for your "Go", then generates detailed sub-tasks. The PRD is re-read from disk before each phase, so any edits you make between steps are picked up.

**Trigger phrases:** "create a plan", "generate tasks from the PRD", "break this down into tasks"

**Output:** `.planning/[project-name]/tasks/tasks-prd-[feature-name].md`

**Task format:**
```
- [ ] 1.0 Parent Task
  - [ ] 1.1 Sub-task
  - [ ] 1.2 Sub-task
- [ ] 2.0 Parent Task
  - [ ] 2.1 Sub-task
```

Also includes a `Relevant Files` section listing every file expected to be created or modified.

---

### `gsdl-execute-plan` — Execute Plan

Works through a task list one sub-task at a time. Marks tasks complete as it goes, commits to git when a parent task finishes, and pauses for your approval before moving to the next sub-task. The task list is always read from disk at the start, so any manual edits you made before execution are respected.

**Trigger phrases:** "execute the plan", "start implementing", "work through the tasks"

**Completion protocol per sub-task:**
1. Implement the sub-task
2. Mark `[ ]` → `[x]` in the task file
3. If all sub-tasks under a parent are done, mark the parent `[x]` and commit to git
4. Pause and wait for your approval to continue

**Permission phrases to continue:** `yes`, `y`, `go`, `continue`, `next`, `proceed`, `keep going`

---

## File Structure

All planning files live under `.planning/`. Implementation code lives at the workspace root.

```
workspace-root/
├── .planning/
│   └── project-name/
│       ├── seed.md
│       └── tasks/
│           ├── prd-feature-name.md
│           └── tasks-prd-feature-name.md
├── src/
│   └── (implementation files)
└── README.md
```

---

## Installation

### Option 1 — `npx skills` (recommended)

Install using the [open agent skills CLI](https://github.com/vercel-labs/skills):

**Install to the current project** (committed with your repo, shared with your team):

```bash
npx skills@1.4.0 add nsantini/gsdl -a cursor
```

**Install globally** (available across all your projects):

```bash
npx skills@1.4.0 add nsantini/gsdl -a cursor -g
```

nb: using `skills@1.4.0` until bug causing not to properly symlic to cursor is fixed

Both commands target Cursor via the `-a cursor` flag. Cursor will automatically detect the skills and make them available to the agent.

### Option 2 — Manual copy

Copy the `.cursor/skills/` folder into your project or `~/.cursor/skills/` for a global install:

```bash
# Project-level
cp -r .cursor/skills/ /path/to/your/project/.cursor/skills/

# Global
cp -r .cursor/skills/ ~/.cursor/skills/
```

---

## Usage

Trigger any skill by describing what you want in natural language in the Cursor chat. The agent will detect the relevant skill and follow the appropriate workflow.

**Run the full pipeline on a new idea:**
> "Let's GSD — I want to build a CLI tool that syncs local files to S3"

**Jump straight to a specific phase:**
> "Create a PRD for the `my-project` seed file"
> "Generate a task list from `.planning/my-project/tasks/prd-auth.md`"
> "Execute the plan at `.planning/my-project/tasks/tasks-prd-auth.md`"
