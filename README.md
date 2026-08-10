# GSDL — Get Shit Done Light

A set of agent skills that run a full engineering workflow end-to-end: from a ticket, a doc, or a
half-formed idea, through a reviewed plan, sliced execution, deterministic gates, a pull request, and
a decisions document with the evidence attached.

**Bring your own stack.** GSDL works with any issue tracker — Linear, Jira, GitHub Issues, Notion —
or none at all, running purely on markdown planning files. It publishes learnings to Slite, Notion,
or Confluence, or nowhere. And it runs on any model provider: steps are routed by *capability tier*
(large / medium / small), not by model name.

Not tied to a single agent tool — skills live under `skills/` and install into Cursor, Claude Code,
or anything else the [open agent skills CLI](https://github.com/vercel-labs/skills) supports.

## Why this exists

GSDL doesn't standardize tools — it standardizes **outputs and handoffs** between stages of getting
work done with an AI agent. *How* you reach a checkpoint doesn't matter (skills, subagents, manual
work). What matters is that the checkpoint's output is consistent every time, and that a human sees
it before the pipeline advances.

## The pipeline

```
1. Pick up the source              (a tracker item, a doc, or a plain idea)
2. Brainstorm / clarify intent
3. Produce a plan            → CHECKPOINT: SPEC.md
4. Decompose into slices     → CHECKPOINT: tracker sub-items + tasks.md
5. Human reviews plan + slices
5.5 Branch pre-flight        (no implementation work on the default branch)
6. Execute via agent or human implementation
7. Verify against deterministic gates → CHECKPOINT: verify-report.md
8. Open PR + address the review bot   → CHECKPOINT
9. Close loop with evidence captured  → CHECKPOINT
```

## Skills

Nine skills implement the pipeline. Use them individually, or chain them end-to-end via the `gsdl`
orchestrator.

| Step | Skill | Tier | Checkpoint output |
|------|-------|------|--------------------|
| 1. Pick up the source | `gsdl-fetch-source` | SMALL | project + `seed.md` created, provider config resolved |
| 2. Brainstorm / clarify | *(folded into steps 1 & 3)* | LARGE | — |
| 3. Produce a plan | `gsdl-create-spec` | LARGE | `.planning/[project]/SPEC.md` |
| 4. Decompose into slices | `gsdl-decompose` | LARGE | tracker sub-items + `.planning/[project]/tasks.md` |
| 4b. Tracker writes | `gsdl-tracker-sync` | SMALL | sub-items created / commented / transitioned |
| 5. Human review | *(gate — no skill; enforced by the orchestrator)* | — | — |
| 5.5. Branch pre-flight | *(orchestrator; guarded again in `gsdl-execute`)* | any | working branch created, never `main` |
| 6. Execute | `gsdl-execute` | MEDIUM | commits + tracker sub-item updates |
| 7. Verify gates | `gsdl-verify-gates` | MEDIUM | `.planning/[project]/verify-report.md` |
| 8. Open PR + review bot | `gsdl-open-pr` | MEDIUM | PR open, every bot finding dispositioned |
| 9. Close loop | `gsdl-close-loop` | LARGE | `.planning/[project]/decisions-[project].md`, tracker closed, doc published |

---

## Bring your own stack

Everything external is resolved from one config file, `.planning/gsdl.config.md`, written on the
first run and editable by hand. See [`skills/gsdl/config.md`](skills/gsdl/config.md) for the full
contract.

```markdown
# GSDL Config

## Tracker
- provider: linear          # linear | jira | github | notion | none
- project: ENG
- labels: gsdl
- sub-item-done-state: Done
- parent-review-state: In Review

## Docs
- provider: slite           # slite | notion | confluence | none
- parent: abc123

## Forge
- provider: github          # github | none
- review-bot: auto          # auto | none | <bot login>

## Models
- provider: anthropic       # anthropic | openai | google | other
```

### Issue trackers

| Provider | Sub-items are | Status is |
|---|---|---|
| **Linear** | issues with `parentId` | a workflow state on the team |
| **Jira** | `Sub-task` issues under `fields.parent` | a workflow transition |
| **GitHub Issues** | issues cross-linked to the parent's task list | open / closed (+ optional status label) |
| **Notion** | pages in a tracker database, related to the parent | a `Status` property |
| **none** | lines in `tasks.md` | the checkbox, plus a `worklog.md` entry |

**`none` is a first-class mode, not a degraded one.** The pipeline runs all nine steps, produces every
artifact, and enforces every checkpoint — the tracker's job is just done by markdown files instead.

Access is resolved per provider: connected MCP server first, then API credentials, then a public web
fetch, then a manual paste. Whatever works is recorded in the config so it isn't probed again.

### Documentation targets

The decisions document is always written to disk. Publishing it is optional and goes to **Slite**,
**Notion** (with full markdown-to-block conversion), **Confluence** (storage-format HTML), or nowhere.

### Code forge

Step 8 supports **GitHub** via the `gh` CLI. With `Forge → provider: none` the step is skipped and the
skip is recorded — never silently dropped.

The review bot is not hardcoded. `review-bot: auto` triages findings from whichever bot commented —
Cursor BugBot, CodeRabbit, Copilot, Greptile, and anything else posting as a `[bot]` account — pin one
by login, or set `none` to skip the wait.

---

## Model tiers

GSDL routes each step to a **capability tier**, never to a named model, so the same pipeline runs on
Anthropic, OpenAI, Google, or anything else with more than one model size. See
[`skills/gsdl/models.md`](skills/gsdl/models.md).

| Tier | Used for | Anthropic | OpenAI | Google |
|---|---|---|---|---|
| **LARGE** | Clarifying intent, `SPEC.md`, slicing, the decisions document | Opus | current flagship | Gemini Pro |
| **MEDIUM** | Implementing an approved plan, gates, the PR | Sonnet | current mid-tier | Gemini Flash |
| **SMALL** | Tracker reads/writes, filling templates | Haiku | current small model | Gemini Flash-Lite |

**Large thinks, medium does, small talks to the tracker.** Judgment-heavy steps — whose mistakes
propagate through everything downstream — go to LARGE. Implementing an already-reviewed plan goes to
MEDIUM. Every tracker round-trip goes to SMALL, because none of it is more than "resolve an ID, POST a
mutation".

Model *families* are resolved at run time, so nothing pins a dated model ID. Pin one anyway by setting
`large` / `medium` / `small` explicitly in the config.

**The orchestrator can be invoked from any session model.** You don't need to be on a large model to
run `/gsdl`. It routes each step itself:

- Mechanical steps are **subagents pinned with an explicit model override**.
- Judgment steps run on LARGE: **inline** if your session is already there, otherwise in a **LARGE
  subagent** driven by a *Question Relay* — the subagent returns its clarifying questions, the
  orchestrator puts them to you, and the answers are fed back on the next spawn.
- A step is never downgraded below its tier, and GSDL never asks you to switch models mid-pipeline.
- If your agent tool can't override models at all, everything runs on the session model and the
  intended tier is recorded instead. The checkpoints are the part that must not be skipped.

Delegated steps run **non-interactively** — a subagent that hits something needing a human (no tracker
access, a failing gate, a plan that looks wrong, a bot finding needing a product call) returns a
blocker, and the orchestrator resolves it with you.

---

## The skills in detail

### `gsdl` — Full Pipeline Orchestrator

Runs the whole pipeline end-to-end from any session model. Detects which step a project is at and
resumes from there.

**Trigger phrases:** "let's GSD", "run the GSDL pipeline", "pick up this ticket"

```
/gsdl https://linear.app/team/issue/ENG-123/my-feature   — start from a Linear ticket
/gsdl PROJ-45                                           — start from a Jira issue
/gsdl owner/repo#12                                     — start from a GitHub issue
/gsdl https://myteam.slite.com/p/abc123/My-Spec         — start from a spec doc
/gsdl "a CLI that syncs local files to S3"              — start from an idea, no tracker
/gsdl my-project                                        — resume an existing project
```

A checkpoint is shown between every step and between every parent slice during execution — the
pipeline never auto-advances without confirmation, and Step 5's human review is a real, separate gate
(not folded into Step 4's checkpoint). At each checkpoint you can **edit the artifact directly**; the
next step always re-reads from disk.

**Progress tracking:** `.planning/[project-name]/progress.md` records the current step, the resolved
providers, the working branch, the PR, and active file paths, so `/gsdl [project-name]` always resumes
cleanly — even after a restart or crash.

### `gsdl-fetch-source` — SMALL

Picks up whatever you're starting from — a tracker item, a spec document, or a plain idea — and
creates `.planning/[project-name]/` with `seed.md` populated from it. Also establishes the provider
config on the first run in a workspace.

### `gsdl-create-spec` — LARGE

Generates `SPEC.md`, the single canonical plan document per project (not a per-feature PRD). Asks
clarifying questions first; this is where Step 2's brainstorming concludes. Its Verification Plan
section feeds Step 7.

### `gsdl-decompose` — LARGE (+ SMALL for the writes)

Sizes the decomposition to the work — one slice is a valid answer — then, after an explicit "Go",
creates one tracker sub-item per slice and mirrors them in `tasks.md`. With no tracker, `tasks.md` is
the checkpoint on its own.

### `gsdl-tracker-sync` — SMALL

GSDL's tracker write path across Linear, Jira, GitHub Issues, Notion, and local markdown. It makes no
decisions — callers author the content, this skill applies it — and it owns the invariants (configured
labels, correct parenting, no sprint/cycle assignment, states resolved by name against the real
workflow). If the tracker is unreachable it returns paste-ready text rather than skipping the write.

### `gsdl-execute` — MEDIUM

Works through `tasks.md` one sub-task at a time. **Refuses to write code on the default branch.**
Commits when a parent slice finishes and hands the tracker update to SMALL.

### `gsdl-verify-gates` — MEDIUM

Discovers and runs the project's actual deterministic checks (test/lint/typecheck/build scripts across
JS, Python, Go, Rust, .NET, Ruby, JVM, plus CI config) and produces `verify-report.md`. If no gates
exist, that gap is reported explicitly rather than silently skipped. Hard-stops the pipeline on any
failing gate — a PR is never opened on a red branch.

### `gsdl-open-pr` — MEDIUM

Pushes the branch and opens a PR whose body carries the spec summary, the slices with their tracker
IDs, and the gate table. Then polls for the repo's review bot (every 60s, up to 10 minutes) and
triages each finding: fix it, reject it with a concrete technical reason, or escalate it as needing a
human. Fixes are committed, re-gated, and replied to on the thread. "The bot didn't respond" is
reported as exactly that — never as "no findings".

### `gsdl-close-loop` — LARGE (+ SMALL for the writes)

Captures decisions, architecture changes, the Step 7 gate evidence, and the Step 8 review outcome into
`decisions-[project].md`, then — on confirmation — comments on and transitions the tracker items
(sub-items → done, parent → review, never a terminal state). Optionally publishes the document to
Slite, Notion, or Confluence.

---

## File Structure

```
workspace-root/
├── .planning/
│   ├── gsdl.config.md                   ← providers: tracker, docs, forge, models
│   └── project-name/
│       ├── seed.md
│       ├── progress.md
│       ├── SPEC.md
│       ├── tasks.md
│       ├── worklog.md                   ← only when no tracker is configured
│       ├── verify-report.md
│       └── decisions-project-name.md
├── src/
│   └── (implementation files)
└── README.md
```

Skill source lives at:

```
skills/
├── gsdl/                    (+ config.md, models.md — the shared contracts)
├── gsdl-fetch-source/
├── gsdl-create-spec/        (+ examples.md)
├── gsdl-decompose/
├── gsdl-tracker-sync/
├── gsdl-execute/
├── gsdl-verify-gates/
├── gsdl-open-pr/
└── gsdl-close-loop/
```

---

## Installation

### Option 1 — `npx skills` (recommended)

```bash
# Project-level (committed with your repo, shared with your team)
npx skills@1.4.0 add nsantini/gsdl

# Global (available across all your projects)
npx skills@1.4.0 add nsantini/gsdl -g
```

The CLI detects which agent tools you have and installs to the right place. Pass `-a <agent>` to
target one explicitly when you have several.

nb: pinned to `skills@1.4.0` until a symlink bug in Cursor installs is fixed upstream — drop the pin
if you don't use Cursor.

### Option 2 — Manual copy

Copy `skills/` into wherever your agent looks for skills:

```bash
# Project-level — .cursor/skills/, .claude/skills/, or your agent's equivalent
cp -r skills/ /path/to/your/project/.<agent>/skills/

# Global
cp -r skills/ ~/.agents/skills/
```

---

## Usage

**Run the full pipeline from a ticket:**
> `/gsdl https://linear.app/myteam/issue/ENG-123/my-feature`

**Run it with no tracker at all:**
> `/gsdl "a CLI tool that syncs local files to S3"`

**Resume an in-progress project:**
> `/gsdl my-project`

**Jump straight to a specific step:**
> "Produce SPEC.md for the `my-project` seed file"
> "Decompose `.planning/my-project/SPEC.md` into slices"
> "Verify gates for `my-project`"
> "Open a PR for `my-project` and address the review bot"
> "Close the loop on `my-project`"

### Requirements

Nothing is mandatory except git — every external integration degrades to a documented local path.

- **Issue tracker** — optional. MCP connection or API credentials for Linear, Jira, GitHub Issues, or
  Notion. Without one, `tasks.md` and `worklog.md` are the record.
- **GitHub CLI** (`gh`), authenticated — only for Step 8. Without it, set `Forge → provider: none`.
- **A code review bot** on the repo — optional; if none is installed, Step 8 says so and moves on
  instead of waiting.
- **Docs platform** — optional, only for publishing the decisions document.
