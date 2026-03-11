---
name: gsdl
description: Get Shit Done (GSD) orchestrator. Runs the full project pipeline from idea to implementation: setup → PRD → task list → implementation. Use when the user wants to build something end-to-end, says "let's GSD", "build this from scratch", "get shit done", or wants to run the full development workflow. Coordinates gsdl-setup-project, gsdl-create-prd, gsdl-create-plan, and gsdl-execute-plan skills. Spawns subagents per parent task during implementation to preserve context.
---

# GSD — Get Shit Done Orchestrator

Coordinates the full project pipeline across four phases. Phases 0–2 run **inline** (interactive, in this context). Phase 3 (implementation) **spawns one subagent per parent task** to keep context fresh.

## Pipeline Overview

```
Phase 0: Setup       → gsdl-setup-project SKILL.md (inline)
Phase 1: PRD         → gsdl-create-prd SKILL.md    (inline, interactive Q&A)
Phase 2: Task List   → gsdl-create-plan SKILL.md   (inline, two-phase with "Go")
Phase 3: Implement   → gsdl-execute-plan SKILL.md  (subagent per parent task)
```

---

## Step 1: Detect Current Phase

Before doing anything:

1. Ask the user for the project name if not provided
2. Check what exists in the workspace:

| Condition | Start at |
|-----------|----------|
| No `.planning/[project-name]/` folder | Phase 0 |
| `.planning/[project-name]/seed.md` exists, no `tasks/prd-*.md` | Phase 1 |
| `.planning/[project-name]/tasks/prd-*.md` exists, no `tasks-prd-*.md` | Phase 2 |
| `.planning/[project-name]/tasks/tasks-prd-*.md` exists with unchecked `[ ]` items | Phase 3 |
| All tasks `[x]` | Done |

Announce which phase you're starting from and why before proceeding.

---

## Step 2: Run Each Phase

### Phase 0 — Project Setup (Inline)

Read and follow `~/.cursor/skills/gsdl-setup-project/SKILL.md`.

Complete the full setup (folder, `tasks/`, `seed.md`), then show the checkpoint and wait for user confirmation before continuing to Phase 1.

### Phase 1 — Create PRD (Inline)

Read and follow `~/.cursor/skills/gsdl-create-prd/SKILL.md`.

This phase is interactive — ask clarifying questions, iterate on the PRD with the user, then save it to `.planning/[project-name]/tasks/prd-[feature-name].md`.

Show the checkpoint and wait for confirmation before Phase 2. **Before proceeding to Phase 2, re-read the PRD from disk** at its saved path — the user may have edited the file after reviewing it.

### Phase 2 — Generate Task List (Inline)

Read and follow `~/.cursor/skills/gsdl-create-plan/SKILL.md`.

This has two steps: show parent tasks → wait for user "Go" → generate sub-tasks and save to `.planning/[project-name]/tasks/tasks-prd-[name].md`.

Show the checkpoint and wait for confirmation before Phase 3. **Before proceeding to Phase 3, re-read the task file from disk** at its saved path — the user may have edited the file (reordering, rewording, or adding tasks) after reviewing it.

### Phase 3 — Implementation (Subagent per Parent Task)

**Re-read the task file from disk** to identify all parent tasks (1.0, 2.0, 3.0, ...). Use the on-disk version as the source of truth, not any prior in-context copy.

For each parent task, in order:

1. Read the task file to get the parent task title and its sub-tasks
2. Spawn a `generalPurpose` subagent using the Task tool with this prompt (fill in all `[placeholders]`):

```
Read and follow the skill at: ~/.cursor/skills/gsdl-execute-plan/SKILL.md

You are running in BATCH MODE. Complete ALL sub-tasks under the assigned parent task
without pausing for user approval between sub-tasks. Apply the full completion protocol
after each sub-task (mark [x], update task file, update Relevant Files), but immediately
continue to the next sub-task without waiting.

Project context:
- Project name: [project-name]
- Workspace root: [absolute path to workspace root]
- Task file: [absolute path to .planning/[project-name]/tasks/tasks-prd-*.md]
- Assigned parent task: [N.0] [Parent Task Title]
- Sub-tasks to complete: [N.1] [title], [N.2] [title], ... (list all)

When all sub-tasks under [N.0] are marked [x] and the parent is marked [x]:
- Return a summary: what was built, files created/modified, any issues or blockers
- Do NOT start the next parent task ([N+1].0)
```

3. After the subagent returns, show the user the summary
4. Show the checkpoint and wait for confirmation before spawning the next subagent

---

## Checkpoint Format

Show this between every phase transition and between every parent task in Phase 3:

```
✅ [Phase name / Task N.0 title] complete.

What was done: [1–3 sentence summary]
Files touched: [key files created or modified]

👉 Review the output file before continuing — you can edit it directly and your changes will be picked up automatically.

Ready to continue to [next phase / Task N+1.0]?
Type "yes" / "go" / "next" to continue, or tell me what to review or change first.
```

**Never proceed past a checkpoint without user confirmation. Always re-read the relevant artifact from disk after the user confirms, to pick up any edits they made during the review window.**

---

## Resuming Mid-Pipeline

If GSD is triggered on a project that already has work done:

1. Run phase detection
2. Skip completed phases
3. Announce: "Project '[name]' is at Phase [N]. Resuming from [description]."
4. For Phase 3, check which parent tasks are already `[x]` and skip them

---

## Rules

1. **Always show a checkpoint** between phases and between parent tasks — never chain them automatically
2. **Phases 0–2 are inline** — do not spawn subagents for setup, PRD, or task generation
3. **Each Phase 3 subagent handles exactly one parent task** — never assign two parent tasks to one subagent
4. **Surface blockers immediately** — if a subagent reports an error or missing dependency, pause Phase 3 and resolve with the user before continuing
5. **All files stay within the project folder** — never create files at workspace root
