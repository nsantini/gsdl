---
name: gsdl
description: "GSDL (Get Shit Done Light) orchestrator. Runs the full pipeline: pick up the work item → brainstorm → SPEC.md → decompose into slices → human review → branch → execute → verify gates → open PR and address review-bot findings → close loop with evidence. Use when the user wants to build something end-to-end, says \"let's GSD\", \"run the GSDL pipeline\", \"pick up this ticket\", or wants the full workflow. Coordinates gsdl-fetch-source, gsdl-create-spec, gsdl-decompose, gsdl-execute, gsdl-verify-gates, gsdl-open-pr, gsdl-close-loop, and gsdl-tracker-sync. Works with any issue tracker (Linear, Jira, GitHub Issues, Notion) or none at all, and with any model provider. Can be invoked from ANY session model — it routes each step to that step's capability tier. Accepts a work-item URL/ID, a doc URL, a plain idea, or an existing project name to resume: /gsdl [url|id] OR /gsdl [project-name] OR /gsdl [idea]."
disable-model-invocation: true
---

# GSDL — Get Shit Done Light Orchestrator

Coordinates the full pipeline, from a raw idea or a tracked work item through to a reviewed,
evidenced change:

```
1. Pick up the source     → gsdl-fetch-source     [SMALL]   (tracker item, doc URL, or a plain idea)
2. Brainstorm/clarify     → (folded into Step 1 and Step 3 Q&A)         [LARGE]
3. Produce a plan         → gsdl-create-spec      [LARGE]   → CHECKPOINT: SPEC.md
4. Decompose into slices  → gsdl-decompose        [LARGE]   → CHECKPOINT: slices (tracker sub-items + tasks.md)
5. Human reviews          → (gate — no skill, a real pause before Step 6)
5.5 Branch pre-flight     → (orchestrator, any tier) — never implement on the default branch
6. Execute                → gsdl-execute          [MEDIUM subagent per parent slice]
7. Verify gates           → gsdl-verify-gates     [MEDIUM]  → CHECKPOINT: verify-report.md
8. Open PR + bot review   → gsdl-open-pr          [MEDIUM]  → CHECKPOINT: PR open, findings addressed
9. Close loop             → gsdl-close-loop       [LARGE]   → CHECKPOINT: decisions + evidence, item closed
```

*How* each checkpoint is reached doesn't matter — what matters is that every checkpoint produces a
consistent, reviewable artifact before the pipeline advances. **Never silently skip a checkpoint.**

---

## Provider Configuration

GSDL is not locked to any tracker, docs platform, forge, or model provider. Before Step 1, read the
config contract in [`config.md`](./config.md) (resolve its path the same way you resolve sub-skills,
below) and establish:

- **Tracker** — `linear` \| `jira` \| `github` \| `notion` \| `none`
- **Docs** — `slite` \| `notion` \| `confluence` \| `none`
- **Forge** — `github` \| `none`
- **Models** — which provider's tier mapping to use

Resolution order is: per-project config → workspace config → probe MCP servers and env vars → ask
the user once and **write the answer to `.planning/gsdl.config.md`**. Never ask twice for something
that could have been recorded.

Two provider settings change the pipeline's shape:

| Setting | Effect |
|---|---|
| `Tracker → provider: none` | Step 4 creates no sub-items; `tasks.md` is the work-item list, and tracker comments are appended to `.planning/[project-name]/worklog.md`. Every checkpoint still happens |
| `Forge → provider: none` | Step 8 is skipped; the pipeline goes verify → close-loop. Record the skip in `progress.md` and in the decisions document |

Nothing else in the pipeline changes with the provider.

---

## Model Selection

**This skill can be invoked from any session model.** The orchestrator is a conductor — it routes
each step to its capability tier by spawning a subagent with an explicit model override. The user
never has to switch models mid-pipeline.

| Step | Skill | Tier | Why |
|------|-------|------|-----|
| 1. Pick up the source | `gsdl-fetch-source` | **SMALL** | Fetch content, fill a template — no judgment |
| 2. Brainstorm / clarify | *(inline in Steps 1 & 3)* | **LARGE** | Open-ended intent shaping, tradeoffs, question generation |
| 3. Produce a plan | `gsdl-create-spec` | **LARGE** | Turning fuzzy intent into a reviewable, complete plan |
| 4. Decompose into slices | `gsdl-decompose` | **LARGE** | Slicing strategy and dependency ordering — cheap to get wrong here, expensive downstream |
| 4b. Create the sub-items | `gsdl-tracker-sync` | **SMALL** | Resolving label IDs and POSTing items is mechanical tracker I/O |
| 5. Human review | *(gate)* | — | Human, not model |
| 5.5. Branch pre-flight | *(orchestrator)* | **any** | Three git commands |
| 6. Execute | `gsdl-execute` | **MEDIUM** | Implementation against an already-approved plan |
| 7. Verify gates | `gsdl-verify-gates` | **MEDIUM** | Discovering and running deterministic checks, reporting output verbatim |
| 8. Open PR + bot review | `gsdl-open-pr` | **MEDIUM** | Forge CLI calls, reading review comments, applying scoped fixes |
| 9. Close loop | `gsdl-close-loop` | **LARGE** | Synthesising decisions, architecture changes, and evidence into a durable document |
| 9b. Close in the tracker | `gsdl-tracker-sync` | **SMALL** | Comment + status transition is mechanical tracker I/O |

Tier-to-model mapping for the configured provider is in [`models.md`](./models.md), along with what
to do when the agent tool can't override models at all.

### Routing rules

1. **Never run a step below its tier.** Downgrading silently is the failure mode this section
   exists to prevent. Upgrading is allowed but never required.
2. **Mechanical steps (1, 4b, 6, 7, 8, 9b) are always subagents**, pinned with an explicit model
   override on the spawn call. State the tier and resolved model in the subagent prompt too, so it
   is recorded even where the override isn't honoured.
3. **Judgment steps (2, 3, 4, 9) run on LARGE:**
   - **Session already on a LARGE model** → run them **inline**. Preferred: the Q&A is direct.
   - **Otherwise** → spawn a **LARGE subagent** and use the **Question Relay** below. Do not run
     these steps on a smaller session model, and do not ask the user to switch models.
4. **Subagents never talk to the user.** Anything interactive — a manual paste, a clarifying
   question, gate-failure triage, a bot finding needing a human call — comes back to this context
   as a returned blocker or question, and the orchestrator handles it with the user.
5. **All tracker reads and writes go to SMALL** via `gsdl-tracker-sync`. Never spend a LARGE or
   MEDIUM model on a tracker API round-trip.

### Question Relay (LARGE steps, smaller session model)

The judgment steps are interactive, and subagents can't ask the user anything. Bridge it:

1. Spawn the LARGE subagent with the step's normal prompt, plus:

```
You cannot talk to the user. If you need input before you can produce your artifact,
return exactly:

QUESTIONS
1. [question]
2. [question]

...and stop — do not guess, and do not write the artifact. Otherwise, do the work
and return the artifact path plus a summary.
```

2. If it returns `QUESTIONS`, put them to the user verbatim in this context.
3. Re-spawn the same subagent with the full Q&A appended under a `## Answers so far` heading.
   Repeat until it returns the artifact.
4. Show the artifact as the step's checkpoint as normal.

Keep the relay tight: pass the accumulated Q&A every time, since the subagent starts fresh each spawn.

---

## Step 0: Resolve the Source (Pre-flight)

Before resolving the project name, work out what the user gave you:

| Input | Meaning |
|---|---|
| A tracker item URL or bare ID — `https://linear.app/…/issue/ENG-123/…`, `ENG-123`, `https://…atlassian.net/browse/PROJ-45`, `PROJ-45`, `https://github.com/o/r/issues/12`, `#12` | New project from a tracked work item |
| A document URL — Notion, Slite, Confluence, Google Docs | New project seeded from a spec document |
| An existing project name — `/gsdl eng-123-my-feature` | Resume; skip to Step 2, no fetch |
| A plain description — "a CLI that syncs files to S3" | New project from a local idea, no tracker item |
| Nothing | Ask: "What are we picking up — a ticket, a doc, or an idea?" |

**For anything that isn't a resume**, delegate the fetch to a **SMALL subagent** with this prompt:

```
You are running on the SMALL tier ([resolved model]). Read and follow the gsdl-fetch-source skill
(resolve path per the Skill Path Resolution section of the gsdl orchestrator SKILL.md).

Source: [URL, ID, or the user's description verbatim]
Workspace root: [absolute path to workspace root]

You are running NON-INTERACTIVELY — do not ask the user anything. Resolve the provider config,
create .planning/[project-name]/, and write seed.md from the source.

Return: the project name, the seed.md path, the tracker item ID (or "none"), the item's suggested
git branch name if the tracker exposes one, and a one-line summary of what was fetched.
If you cannot fetch the source (no MCP, no credentials, not publicly shared) or the project already
exists with a seed.md, do NOT guess and do NOT overwrite — return a BLOCKER describing exactly
what's needed, and stop.
```

Then:

1. If the subagent returned a blocker, resolve it **inline with the user in this context** (offer
   the manual paste / MCP setup / overwrite choice per `gsdl-fetch-source`), then re-spawn or
   finish the fetch inline.
2. Record the item's suggested branch name (if any) in `progress.md` as **`Suggested branch`** — it
   must survive to Step 5.5 even across a restart, which is why it is persisted now rather than
   held in context. `seed.md` also carries it under `## Source`.
3. Confirm with the user: *"I picked up [source summary]. I'll use `[project-name]` — does that work?"*
4. Proceed to **Step 2: Detect Current Step**.

---

## Step 1: Resolve Project Name

1. If the user typed `/gsdl my-project` (resume), use `my-project` directly.
2. If a source was resolved in Step 0, use the name it returned.
3. Normalize to kebab-case. Prefix with the tracker item ID when there is one
   (`eng-123-rule-calculator`); otherwise use the title alone (`rule-calculator`).

---

## Step 2: Detect Current Step

Read `.planning/[project-name]/progress.md` if it exists.

If present, extract:
- `step` — last recorded step
- `spec` — path to `SPEC.md`
- `tasks` — path to `tasks.md`
- `verify_report` — path to `verify-report.md`
- `item` — the tracker item ID, or `none`
- `suggested_branch` — the tracker's suggested branch name from Step 0
- `branch` — the working branch for this project, once created
- `pr` — the pull request URL, or `skipped`

**Verify** the recorded files still exist before trusting them; fall back to a filesystem scan if not.

If `progress.md` is missing or stale, scan the filesystem:

| Condition | Start at |
|-----------|----------|
| No `.planning/[project-name]/` folder | Step 0 (fetch source) |
| `seed.md` exists, no `SPEC.md` | Step 3 (spec) |
| `SPEC.md` exists, no `tasks.md` | Step 4 (decompose) |
| `tasks.md` exists with unchecked `[ ]` items | Step 5.5 (branch), then Step 6 (execute) |
| All tasks `[x]`, no `verify-report.md` | Step 7 (verify) |
| `verify-report.md` exists (all gates pass), no PR recorded | Step 8 (open PR), or Step 9 if `Forge → provider: none` |
| PR recorded or skipped, no `decisions-*.md` | Step 9 (close) |
| `decisions-[project-name].md` exists | Done |

Announce which step you're starting from and why before proceeding.

---

## Skill Path Resolution

When reading a sub-skill or a shared reference file, resolve its path in this priority order:

1. `~/.agents/skills/[skill-name]/` — global install via `npx skills` (preferred)
2. Project-local install matching the current agent, e.g. `.cursor/skills/[skill-name]/` or
   `.claude/skills/[skill-name]/`
3. `skills/[skill-name]/` relative to this repo — running directly from source, not installed

Use whichever exists first. The shared references `config.md` and `models.md` live alongside this
file, in the `gsdl` skill's own directory.

Sub-skills used by this orchestrator:

- `gsdl-fetch-source` — picks up the work item / doc / idea, creates the project + `seed.md` (Step 1) — **SMALL subagent**
- `gsdl-create-spec` — generates `SPEC.md` (Step 3) — **LARGE**
- `gsdl-decompose` — breaks `SPEC.md` into slices (Step 4) — **LARGE**
- `gsdl-tracker-sync` — all tracker writes: sub-item creation, comments, status transitions — **SMALL subagent**
- `gsdl-execute` — implements slices (Step 6) — **MEDIUM subagent per parent slice**
- `gsdl-verify-gates` — runs deterministic checks (Step 7) — **MEDIUM subagent**
- `gsdl-open-pr` — opens the PR and addresses review-bot comments (Step 8) — **MEDIUM subagent**
- `gsdl-close-loop` — captures decisions + evidence, closes the tracker item, publishes docs (Step 9) — **LARGE**

Each skill declares its tier in its own frontmatter (`tier:`) and a `## Tier` section — those are
the source of truth if this table ever drifts.

---

## Running Each Step

### Step 3 — Produce a Plan → CHECKPOINT: SPEC.md (LARGE)

Read and follow `gsdl-create-spec`. Interactive — ask clarifying questions, iterate with the user,
save `SPEC.md`. This is where Step 2's brainstorming concludes.

- Session on a LARGE model → run inline.
- Otherwise → LARGE subagent + **Question Relay**.

Update `progress.md` (step `spec`, `spec` path). Show the checkpoint and wait for confirmation.
**Before Step 4, re-read `SPEC.md` from disk** — the user may have edited it.

### Step 4 — Decompose into Slices → CHECKPOINT: slices (LARGE + SMALL)

Read and follow `gsdl-decompose`. Two-phase: parent slices → wait for "Go" → sub-tasks + tracker
sub-items + `tasks.md`.

- **Slicing (Phase 1 and the sub-task breakdown)** is LARGE: inline if the session is on a LARGE
  model, otherwise a LARGE subagent with Question Relay.
- **Creating the sub-items in the tracker (Phase 2's API work)** is delegated to a **SMALL
  subagent** running `gsdl-tracker-sync` — hand it the resolved slice titles, descriptions, parent
  item ID and project/team ID, and let it resolve labels and create the items.
- **With `Tracker → provider: none`**, there is no 4b: `tasks.md` is the checkpoint artifact on its
  own, and the slices are reviewed there.

Update `progress.md` (step `decompose`, `tasks` path). Show the checkpoint listing the created
sub-item links (or the local slice list when there is no tracker).

### Step 5 — Human Reviews Plan + Slices (Gate, no skill)

**This is the most important gate in the pipeline — do not fold it into Step 4's checkpoint.**
Explicitly ask the user to confirm they've reviewed both `SPEC.md` and the slices before execution
starts:

```
👉 Before I start implementation: please review SPEC.md and the slices.
Ready for me to start executing? (yes/go to proceed, or tell me what to change first)
```

Only proceed to Step 5.5 on explicit confirmation.

### Step 5.5 — Branch Pre-flight (Orchestrator, any tier)

**No implementation work happens on the default branch.** Run this before spawning the first
execute subagent, and re-check it on every resume that lands at Step 6.

```bash
git rev-parse --abbrev-ref HEAD
git status --porcelain
git remote show origin | sed -n 's/.*HEAD branch: //p'   # or: gh repo view --json defaultBranchRef
```

Decide:

| Situation | Action |
|-----------|--------|
| On the default branch (`main`/`master`) | Create the working branch and switch to it |
| On a non-default branch that matches this project's recorded `branch` | Continue on it |
| On a different non-default branch | **Stop and ask the user** — do not switch or create silently; they may be mid-work on something else |
| Uncommitted changes present | Report them to the user before creating the branch; let them decide (commit / stash / continue) |
| Not a git repository | Report it and ask whether to `git init` or continue without version control — Steps 6 and 9 both read git history |

Branch name, in priority order:

1. `progress.md`'s **`Suggested branch`** — the tracker's suggested name, recorded at Step 0 (e.g.
   `nico/eng-441-rule-calculator`). On a resume, read it from disk; falling back to the project name
   because it isn't in context would give the same item two different branches depending on entry
   path. `seed.md`'s `## Source` carries the same value if `progress.md` predates this field.
2. Otherwise the project name, which is already kebab-case (e.g. `eng-441-rule-calculator`).

```bash
git checkout -b [branch-name]
```

Record `branch` in `progress.md`, then tell the user which branch the work will land on before
Step 6 starts.

### Step 6 — Execute (MEDIUM Subagent per Parent Slice)

Update `progress.md` to step `execute` before starting.

**Re-read `tasks.md` from disk** to identify all parent slices. For each, in order:

1. Read the task file to get the parent slice title, its sub-tasks, and its tracker sub-item tag
2. Spawn a subagent with the **MEDIUM** tier model override and this prompt (fill in all
   `[placeholders]`):

```
You are running on the MEDIUM tier ([resolved model]). Read and follow the gsdl-execute skill
(resolve path per the Skill Path Resolution section of the gsdl orchestrator SKILL.md).

You are running in BATCH MODE. Complete ALL sub-tasks under the assigned parent slice
without pausing for user approval between sub-tasks. Apply the full completion protocol
after each sub-task (mark [x], update tasks.md), but immediately continue to the next
sub-task without waiting.

Project context:
- Project name: [project-name]
- Workspace root: [absolute path to workspace root]
- Task file: [absolute path to .planning/[project-name]/tasks.md]
- Working branch: [branch-name] — already created; verify you are on it before writing code,
  and STOP with a blocker if you are on the default branch
- Assigned parent slice: [N.0] [Parent Slice Title] [tracker sub-item tag, or "no tracker"]
- Sub-tasks to complete: [N.1] [title], [N.2] [title], ... (list all)

When all sub-tasks under [N.0] are marked [x] and the parent is marked [x]:
- Commit to git per gsdl-execute's Parent Slice Completion protocol
- Return the tracker sub-item update payload (comment text + target state) rather than calling
  the tracker yourself — the orchestrator dispatches it to a SMALL subagent
- Return a summary: what was built, files created/modified, any issues or blockers
- Do NOT start the next parent slice ([N+1].0)
```

3. Dispatch the returned payload to a **SMALL subagent** running `gsdl-tracker-sync` (comment on the
   sub-item + move it to the configured done state). With `Tracker → provider: none`, the same
   payload is appended to `worklog.md` instead — the record still gets written.
4. Show the user the summary
5. Show the checkpoint and wait for confirmation before spawning the next subagent

### Step 7 — Verify Against Deterministic Gates → CHECKPOINT: verify-report.md (MEDIUM Subagent)

After all parent slices are `[x]`, update `progress.md` to step `verify`.

Spawn a subagent with the **MEDIUM** tier model override and this prompt:

```
You are running on the MEDIUM tier ([resolved model]). Read and follow the gsdl-verify-gates skill
(resolve path per the Skill Path Resolution section of the gsdl orchestrator SKILL.md).

Project context:
- Project name: [project-name]
- Workspace root: [absolute path to workspace root]
- Spec: [absolute path to SPEC.md]
- Task file: [absolute path to tasks.md]

You are running NON-INTERACTIVELY — do not ask the user anything, and do not attempt to fix
failing gates. Discover the project's real deterministic checks, run them, and write
verify-report.md with the command output captured verbatim.

Return: the report path, each gate's pass/fail status, and the verbatim output of any failure.
```

**Gate failures are triaged by the orchestrator, not by the subagent.** If any gate fails, **stop
the pipeline here** — surface the failure to the user, help resolve it (spawning a fresh MEDIUM
`gsdl-execute` subagent for the fix if needed), re-run this step, and do not advance until all
gates pass.

Update `progress.md` (step `verify`, `verify_report` path) once all gates pass.

### Step 8 — Open PR + Address Review Bot → CHECKPOINT (MEDIUM Subagent)

**Skip this step entirely when `Forge → provider: none`** — record `PR: skipped` in `progress.md`,
tell the user the pipeline is going straight to close-loop, and move to Step 9.

Otherwise update `progress.md` to step `pr` and spawn a subagent with the **MEDIUM** tier model
override:

```
You are running on the MEDIUM tier ([resolved model]). Read and follow the gsdl-open-pr skill
(resolve path per the Skill Path Resolution section of the gsdl orchestrator SKILL.md).

Project context:
- Project name: [project-name]
- Workspace root: [absolute path to workspace root]
- Working branch: [branch-name]
- Tracker item: [ITEM-ID] — [item URL], or "none"
- Spec: [absolute path to SPEC.md]
- Task file: [absolute path to tasks.md]
- Verify report: [absolute path to verify-report.md]
- Review bot: [config value — auto | none | <login>]
- Existing PR (if resuming): [PR URL or "none"]

You are running NON-INTERACTIVELY — do not ask the user anything. Push the branch, open the PR
(or reuse the existing one), then poll for review-bot comments and address the ones
that are in scope.

Return: the PR URL, every bot finding with its disposition (fixed / rejected-with-reason /
needs-human), and whether anything is still outstanding. Escalate as a BLOCKER anything that
needs a product decision, changes SPEC.md's agreed scope, or that you could not fix.
```

Then:

1. Record the PR URL in `progress.md`.
2. Show the user the PR link and the findings table.
3. Any finding marked **needs-human** is resolved with the user in this context, then a fresh
   MEDIUM subagent applies the agreed fix and re-runs Step 7's gates on it.
4. Checkpoint: do not advance to Step 9 until the user confirms the PR is in the state they want.

### Step 9 — Close Loop with Evidence Captured → CHECKPOINT (LARGE)

Update `progress.md` to step `close`.

Read and follow `gsdl-close-loop`, passing:
- `PROJECT_NAME`, `WORKSPACE_ROOT`
- `SPEC_PATH`, `TASKS_PATH`, `VERIFY_REPORT_PATH` (all recorded in `progress.md`)
- `PR_URL` and the findings disposition table from Step 8 (or `skipped`)

Session on a LARGE model → run inline. Otherwise → LARGE subagent with Question Relay; the tracker
closure itself is dispatched to a **SMALL subagent** (`gsdl-tracker-sync`) either way.

The skill generates the decisions+evidence document, then closes the tracker item on the user's
confirmation, and optionally publishes to the configured docs provider.

After the document is saved and the user has answered the closure and publish prompts, update
`progress.md` to step `done`, then show:

```
🎉 Project '[project-name]' is complete.

Decisions & Evidence: .planning/[project-name]/decisions-[project-name].md
Verify report: .planning/[project-name]/verify-report.md
Pull request: [PR URL, or "skipped — no forge configured"]
Tracker: [closed item links, or "local only — see worklog.md"]
[If published: Published to [provider]: [URL]]

Run /gsdl [project-name] again if you want to resume or extend this project.
```

---

## Progress File

Maintain `.planning/[project-name]/progress.md` throughout. Write it after every step transition.

### Format

```markdown
# [project-name] — GSDL Progress

## State

- Step: [source | spec | decompose | execute | verify | pr | close | done]
- Last Updated: [YYYY-MM-DD]

## Providers

- Tracker: [linear | jira | github | notion | none]
- Docs: [slite | notion | confluence | none]
- Forge: [github | none]
- Models: [provider name] (LARGE: [model], MEDIUM: [model], SMALL: [model])

## Active Files

- SPEC: [relative path to SPEC.md, or "none"]
- Tasks: [relative path to tasks.md, or "none"]
- Verify Report: [relative path to verify-report.md, or "none"]
- Item: [tracker item ID, or "none"]
- Suggested branch: [tracker's suggested branch name from Step 0, or "none"]
- Branch: [working branch name once created, or "none"]
- PR: [pull request URL, "skipped", or "none"]
```

`Suggested branch` is written at Step 0 and never cleared — Step 5.5 reads it from here, including
on a resume in a fresh context. The `## Providers` block is a snapshot for the record; the config
file remains the source of truth.

### When to write/update

| Event | Step value | SPEC | Tasks | Verify Report | Suggested branch | Branch | PR |
|-------|-----------|------|-------|----------------|------------------|--------|----|
| Step 0 complete (source picked up, project created) | `spec` | none | none | none | name or none | none | none |
| `SPEC.md` saved | `decompose` | path | none | none | carried | none | none |
| `tasks.md` + sub-items created | `execute` | path | path | none | carried | none | none |
| Branch created (Step 5.5) | `execute` | path | path | none | carried | name | none |
| All slices complete | `verify` | path | path | none | carried | name | none |
| All gates pass | `pr` | path | path | path | carried | name | none |
| PR open and reviewed, or step skipped | `close` | path | path | path | carried | name | url \| skipped |
| Decisions doc saved + tracker closed | `done` | path | path | path | carried | name | url \| skipped |

---

## Checkpoint Format

Show this between every step transition and between every parent slice in Step 6:

```
✅ [Step name / Slice N.0 title] complete.

What was done: [1–3 sentence summary]
Files touched: [key files created or modified]

👉 Review the output before continuing — you can edit it directly and your changes will be picked up automatically.

Ready to continue to [next step / Slice N+1.0]?
Type "yes" / "go" / "next" to continue, or tell me what to review or change first.
```

**Never proceed past a checkpoint without user confirmation. Always re-read the relevant artifact
from disk after the user confirms.**

---

## Resuming Mid-Pipeline

1. Read `progress.md` → verify files on disk → fall back to filesystem scan if needed
2. Skip completed steps
3. Announce: "Project '[name]' is at Step [N] ([step name]). Resuming from [description]."
4. **Re-run Step 5.5's branch check** on any resume that will write code — the user may be on a
   different branch than when they left
5. For Step 6, check which parent slices are already `[x]` and skip them
6. For Step 8, if a PR is already recorded, reuse it — push new commits and re-poll the review bot
   rather than opening a second PR
7. For Step 9, if `decisions-[project-name].md` already exists, report the project is done — do not
   re-run

---

## Rules

1. **Providers come from config, never from assumption** — resolve once per `config.md`, record the
   answer, and never silently switch trackers, docs targets, or forges between runs
2. **The orchestrator runs on any model** — never tell the user to switch models; route each step to
   its tier per Model Selection, and never downgrade a step below its tier
3. **Always show a checkpoint** between steps and between parent slices — never chain them
   automatically
4. **All tracker I/O goes to SMALL** via `gsdl-fetch-source` / `gsdl-tracker-sync` — no tracker API
   round-trips on MEDIUM or LARGE
5. **Steps 1, 4b, 6, 7, 8, 9b are subagents** — always pass the tier override on the spawn; a
   subagent that returns a blocker gets handled with the user in this context, never re-delegated blind
6. **Step 6's subagent handles exactly one parent slice** — never assign two slices to one subagent
7. **Step 5 is a real, separate gate** — do not let the user "yes" through Step 4's checkpoint
   straight into execution without an explicit review confirmation
8. **Never write code on the default branch** — Step 5.5 runs before the first execute subagent, and
   again on every resume
9. **Step 7 is a hard stop on failure** — never advance with a failing gate
10. **Step 8 opens exactly one PR per project** — resuming reuses it; skipping it requires
    `Forge → provider: none` and is always recorded, never silent
11. **Surface blockers immediately** — if a subagent reports an error, pause and resolve with the
    user before continuing
12. **All planning files stay within the project folder** — never create files at workspace root
    (the shared `gsdl.config.md` lives at `.planning/`, not the repo root)
13. **Keep `progress.md` current** — update at every step transition so `/gsdl [project-name]` always
    resumes cleanly
14. **Step 9's tracker closure and doc publishing are the only optional parts** — the decisions+evidence
    document itself is mandatory
