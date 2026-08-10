---
name: gsdl-execute
description: Execute tasks.md step-by-step with completion tracking, committing to git and updating the corresponding tracker sub-item as each slice completes. Use when the user wants to work through a plan, implement tasks sequentially, or asks to "execute the plan" or "start implementing". This is Step 6 (Execute) of the GSDL pipeline. Runs on the MEDIUM tier.
tier: medium
disable-model-invocation: true
---

# Execute

Guides implementation of `tasks.md` with structured completion tracking. This is **Step 6 (Execute
via agent or human implementation)**:
`Pick up source → Brainstorm → SPEC.md → Slices → Review → Branch → Execute → Verify → PR + bot → Close loop`

Step 6 only begins after the Step 5 human-review gate — the caller (a human or the `gsdl`
orchestrator) is responsible for confirming the slices were reviewed before invoking this skill.

## Tier

**MEDIUM.** By the time this skill runs, the hard thinking is already done and reviewed: `SPEC.md`
and the sliced `tasks.md` are the brief. This step is implementation against an approved plan. When
the `gsdl` orchestrator runs Step 6 it spawns one MEDIUM subagent per parent slice.

**If the plan itself looks wrong** — a slice contradicts `SPEC.md`, a sub-task is impossible as
written, or the approach needs rethinking — stop and escalate to the caller rather than redesigning.
Plan changes belong back in Step 3/4 on the LARGE tier.

**Tracker writes are not yours to make.** Comments and status transitions go to a SMALL subagent
running `gsdl-tracker-sync`. Under the orchestrator, return the payload and let it dispatch;
standalone, spawn the SMALL subagent yourself.

## Branch Pre-flight — Before Any Code Is Written

**Never implement on the default branch.** Before touching a file:

```bash
git rev-parse --abbrev-ref HEAD
git remote show origin | sed -n 's/.*HEAD branch: //p'   # or: gh repo view --json defaultBranchRef
```

| Situation | Action |
|-----------|--------|
| On the branch the caller told you to use | Continue |
| On the default branch (`main`/`master`) | **Stop.** As a subagent, return a BLOCKER — the orchestrator's Step 5.5 owns branch creation. Standalone, create the branch yourself using the **same priority Step 5.5 uses** — the tracker's suggested branch name from `seed.md`'s `## Source` if present, otherwise the project name — switch to it, and tell the user |
| On some other non-default branch | **Stop and ask** (or return a BLOCKER) — never switch branches on the user's behalf |
| Not a git repository | Return a BLOCKER — Step 6's commits and Step 9's history analysis both need one |

Re-check this after any interruption or resume; do not assume the branch you started on is still the
one you're on.

## Project Context

Task lists live at `.planning/[project-name]/tasks.md`. Each parent slice may be tagged with its
tracker sub-item identifier (e.g., `1.0 [ENG-124] ...`, `1.0 [PROJ-45] ...`, `1.0 [#128] ...`). **An
untagged slice means the project has no tracker** — that is normal, and the completion protocol below
still applies with the record going to `worklog.md` instead. Implementation code lives at the
workspace root, not inside `.planning/`.

## Operating Modes

### Standard Mode (default)

Work on **ONE sub-task at a time**. Do **NOT** start the next sub-task until the user explicitly
approves. Stop after each sub-task and wait for "yes", "go", "next", etc.

### Batch Mode (gsdl orchestrator only)

If instructed **"BATCH MODE"**, complete **all sub-tasks under your assigned parent slice** without
pausing between sub-tasks. Apply the full completion protocol after each, then continue immediately.
When the whole parent slice is `[x]`, follow **Parent Slice Completion**, then **stop and return a
summary** — do not start another parent slice.

## Completion Protocol

### When You Finish a Sub-Task

1. **Mark it `[x]`** in `tasks.md` immediately and save
2. **Check the parent**: if all its sub-tasks are `[x]`, mark the parent `[x]` too and run **Parent
   Slice Completion** below
3. **Pause** (Standard Mode only) and wait for approval before the next sub-task. Batch Mode
   continues immediately.

### Parent Slice Completion

Once all sub-tasks under a parent slice are `[x]` and the parent itself is marked `[x]`:

1. **Stage and commit**:

```bash
git add -A
git commit -m "$(cat <<'EOF'
[N.0] [Parent Slice Title]

- Bullet summarizing sub-task 1
- Bullet summarizing sub-task 2
EOF
)"
```

2. **Update the tracker sub-item** tagged on this parent slice. Write the comment body — commit hash,
   files touched, what was implemented — but hand the API call to **SMALL** via `gsdl-tracker-sync`
   (Operation B), with `TARGET_STATE` set to the configured `sub-item-done-state` (default `Done`).
   Sub-items go straight to done when their slice is complete: they are not individually code-reviewed,
   the parent item is the review unit (it moves to review at close-loop).
   - **Under the orchestrator**: return the payload (item ID, comment body, target state) instead of
     calling the tracker yourself — the orchestrator dispatches it.
   - **Standalone**: spawn a SMALL subagent running `gsdl-tracker-sync` with that payload.
   - **No tracker configured**: the same payload goes to `.planning/[project-name]/worklog.md` via
     `gsdl-tracker-sync`'s `none` path. The record is still written.
   - **Tracker unreachable**: `gsdl-tracker-sync` returns a BLOCKER with paste-ready text — surface
     it in the checkpoint summary so the user can apply it manually. Never skip the update silently.
3. **Then** pause for user approval (Standard Mode) or continue to the next parent slice (Batch Mode).

### Example

```markdown
Before:
- [ ] 1.0 [ENG-124] Parent Slice
  - [x] 1.1 First sub-task
  - [ ] 1.2 Second sub-task (just finished)
  - [ ] 1.3 Third sub-task

After (1.2 done):
- [ ] 1.0 [ENG-124] Parent Slice
  - [x] 1.1 First sub-task
  - [x] 1.2 Second sub-task
  - [ ] 1.3 Third sub-task (next up)

After (1.3 done, parent complete → commit + tracker comment):
- [x] 1.0 [ENG-124] Parent Slice
  - [x] 1.1 First sub-task
  - [x] 1.2 Second sub-task
  - [x] 1.3 Third sub-task
```

## Task List Maintenance

- Mark completed items `[x]` as you go
- Add newly discovered tasks/sub-tasks if more work surfaces
- Keep the "Relevant Files" section current — every file created or modified, implementation + tests,
  with a one-line purpose each

## Implementation Process

### Before Starting Work

1. **Run the Branch Pre-flight above** — confirm you are not on the default branch.
2. **Read `tasks.md` from disk** — never rely on an in-context version; the user may have edited it
   (reordered, reworded, added tasks) since decomposition.
3. **Identify the next unchecked `[ ]` sub-task**
4. **Check dependencies**: ensure prior tasks are done
5. **Understand the goal** before writing code

### During Implementation

1. Focus only on the current sub-task
2. Implement thoroughly — code, tests, and docs as needed, driven by the Verification Plan in `SPEC.md`
3. Verify your own work locally before marking it done (this is not a substitute for Step 7's formal
   gate run — it's just not leaving obviously broken code for the next sub-task)

### After Completing a Sub-Task

1. Mark `[x]` in `tasks.md` and save
2. If parent complete, run **Parent Slice Completion** (commit + tracker update)
3. Update "Relevant Files" with any new files
4. Report to the user what was completed
5. Ask "Ready to move to the next sub-task?" and wait (Standard Mode)

## User Permission Phrases

Accept as permission to continue: `yes`, `y`, `go`, `continue`, `next`, `proceed`, `keep going`

Do NOT continue if the user asks questions, requests changes, wants to review something, or says
"wait"/"hold on"/"stop".

## Handling Changes and Additions

### Discovered Issues
Add a new task/sub-task, tell the user, ask if they want it addressed now or later.

### Task Modifications
Update `tasks.md` as requested, confirm with the user, resume from the current position.

### Skipping Tasks
Mark with a note (`- [ ] 2.3 [SKIPPED] Original description`), continue, update the parent status
appropriately.

## When All Slices Are Complete

Once every parent slice in `tasks.md` is `[x]`, tell the user implementation (Step 6) is done and the
next step is **`gsdl-verify-gates`** (Step 7) — do not proceed to verification, PR creation, or
close-out yourself; each is a separate, deliberate step.

## File Location

- **Path**: `.planning/[project-name]/tasks.md`
- **Example**: `.planning/eng-124-user-authentication/tasks.md`
