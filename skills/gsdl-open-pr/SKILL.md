---
name: gsdl-open-pr
description: "Push the working branch, open a GitHub pull request for the project, then wait for the repo's automated code-review bot (Cursor BugBot, CodeRabbit, Copilot, Greptile, or whichever is installed) and address its findings. Use when implementation and gates are done and the change needs a PR, or when the user asks to open a PR, check bot review comments, or address automated review feedback. This is Step 8 (Open PR + Address Review Bot) of the GSDL pipeline. Runs on the MEDIUM tier."
tier: medium
disable-model-invocation: true
---

# Open PR + Address the Review Bot

Turns a verified branch into a reviewed pull request. This is **Step 8**:
`Pick up source → Brainstorm → SPEC.md → Slices → Review → Branch → Execute → Verify → PR + bot → Close loop`

The point of this step is that **automated review happens before a human is asked to look**. A review
bot catches a class of defect the deterministic gates in Step 7 can't — logic errors, missed edge
cases, unsafe assumptions — and it is cheap to address them before a reviewer spends attention on them.

**Forge support**: GitHub, via the `gh` CLI. With `Forge → provider: none` in the config, this step
does not run at all — the orchestrator skips from Step 7 to Step 9 and records the skip.

**Bot support is not hardcoded.** `Forge → review-bot` in the config selects it:

| Value | Behaviour |
|---|---|
| `auto` (default) | Triage findings from whichever review bot commented on the PR |
| `<login>` | Only treat that account's comments as bot findings (e.g. `cursor[bot]`) |
| `none` | Open the PR and return immediately — no polling, no triage |

## Tier

**MEDIUM.** Pushing a branch, calling `gh`, reading review comments, and applying scoped fixes to
code that already has an approved spec. The judgment ceiling here is "is this finding real and in
scope?" — anything above that ceiling is escalated to the caller rather than decided here.

**Non-interactive when run as a subagent**: don't ask the user anything. Escalate as a BLOCKER
anything that needs a product decision, contradicts `SPEC.md`, or that you couldn't fix.

## Prerequisites

1. **`verify-report.md` exists with all gates passing** (Step 7). If a gate is failing, stop — do not
   open a PR on a red branch.
2. **All parent slices in `tasks.md` are `[x]`**.
3. **`gh` is authenticated**: `gh auth status`. If not, return a BLOCKER — do not fall back to
   opening a PR by URL in a browser.

## Inputs

- `PROJECT_NAME`, `WORKSPACE_ROOT`
- `BRANCH` — the working branch
- `ITEM` — tracker item ID + URL, or `none`
- `SPEC_PATH`, `TASKS_PATH`, `VERIFY_REPORT_PATH`
- `REVIEW_BOT` — `auto`, `none`, or a specific login
- `EXISTING_PR` — a PR URL if resuming, otherwise `none`

---

## Step 1 — Pre-flight

```bash
git rev-parse --abbrev-ref HEAD
git status --porcelain
gh repo view --json nameWithOwner,defaultBranchRef
```

- **On the default branch** → BLOCKER. This step never opens a PR from `main`.
- **Uncommitted changes** → BLOCKER, listing the files. Step 6 should have committed everything;
  unexpected working-tree changes mean something is out of sync.
- **Branch has no commits ahead of the default branch** (`git log --oneline origin/[default]..HEAD`)
  → BLOCKER, nothing to PR.

## Step 2 — Push and Open the PR

```bash
git push -u origin [BRANCH]
```

If `EXISTING_PR` is set, or `gh pr view --json url` already resolves for this branch, **reuse it** —
push the new commits and skip to Step 3. Never open a second PR for a project.

Otherwise:

```bash
gh pr create --base [default-branch] --head [BRANCH] --title "[ITEM-ID] [Item title]" --body "$(cat <<'EOF'
[body]
EOF
)"
```

With no tracker item, title the PR from `SPEC.md`'s overview instead of a bare item ID.

### PR body template

```markdown
## Summary

[2–4 sentences from SPEC.md's Overview and Goals — what this change does and why]

[Tracker: [ITEM-ID]([item URL]) — omit this line entirely when there is no tracker item]

## What's in this PR

- [N.0] [Parent slice title] — [item ID, if any]
- [N.0] [Parent slice title] — [item ID, if any]

## Verification

| Gate | Command | Result |
|------|---------|--------|
| ... | ... | ✅ Pass |

Full report: `.planning/[PROJECT_NAME]/verify-report.md`

## Out of scope

[SPEC.md's Non-Goals, condensed]
```

Pull the gate table straight from `verify-report.md` — do not re-run gates here.

Record the returned PR URL and number.

## Step 3 — Wait for the Review Bot

**Skip this step entirely when `REVIEW_BOT` is `none`** — report the PR URL and return.

Review bots run asynchronously; they are normally not present the second the PR opens.

**Poll every 60 seconds, up to 10 minutes.** Between polls, do nothing else — don't start speculative
work.

```bash
# top-level PR comments and reviews
gh pr view [PR_NUMBER] --json comments,reviews,latestReviews

# inline review comments (bot findings usually land here)
gh api repos/[owner]/[repo]/pulls/[PR_NUMBER]/comments

# bots often register as a check too
gh pr checks [PR_NUMBER]
```

**Identifying the bot:**

- `REVIEW_BOT: <login>` → match that login, case-insensitively, and nothing else.
- `REVIEW_BOT: auto` → treat as a review bot any comment author whose login ends in `[bot]`, or
  matches one of `cursor`, `coderabbit`, `copilot`, `greptile`, `sonarcloud`, `codacy`,
  `deepsource`, `sourcery`, `qodo`, `ellipsis` — or whose body identifies itself as an automated
  review. Ignore human reviewers: this step does not triage a teammate's comments.

**Record the author login you matched**, so the report says exactly whose findings these are. If more
than one bot commented, triage all of them and attribute each finding.

Stop polling early once a bot has posted a review (a summary comment, inline comments, or a completed
check).

**If nothing arrives within 10 minutes**: that is a reportable outcome, not a failure and not a pass.
Return "no review bot responded within 10 minutes — PR is open at [URL], findings unknown" and let the
caller decide whether to wait longer or proceed. Never report "no findings" when what actually
happened is "no response".

## Step 4 — Triage Each Finding

For every bot comment, classify it:

| Class | Meaning | Action |
|-------|---------|--------|
| **Fix** | A real defect in code this project touched, fixable within `SPEC.md`'s agreed scope | Fix it |
| **Reject** | A false positive, or correct-as-written (intentional pattern, covered elsewhere, misread of the code) | Don't change code; reply with the reason |
| **Needs-human** | Real, but needs a product/architecture decision, changes agreed scope, or touches code outside this project | Escalate — don't fix, don't dismiss |
| **Out of scope** | About pre-existing code this PR didn't touch | Reply noting it's pre-existing; suggest a follow-up item |

Be honest in both directions: **do not fix a finding you believe is wrong just to clear the list**,
and do not reject a finding because fixing it is inconvenient. A rejection must carry a concrete
technical reason.

## Step 5 — Apply Fixes

For findings classed **Fix**:

1. Make the change — minimal and targeted to the finding, no opportunistic refactoring
2. **Re-run the affected gates** from `verify-report.md` (at minimum lint/typecheck/tests for the
   touched area) — a bot fix that breaks a gate is worse than the finding
3. Stage and commit them together — **`git add` first**, or the commit lands empty and the PR keeps
   the defect:

```bash
git add -A
git commit -m "$(cat <<'EOF'
Address review bot findings

- [file:line] [what was wrong → what changed]
- [file:line] [what was wrong → what changed]
EOF
)"
```

4. `git push` — then confirm the commit actually carries the changes (`git show --stat HEAD`); an
   empty or stale commit means the fixes never made it to the PR
5. Update `.planning/[PROJECT_NAME]/verify-report.md` with the re-run gate results, noting they follow
   the bot fixes

## Step 6 — Reply on the PR

Reply to each finding so the thread shows a disposition — a reviewer should never have to guess
whether a bot comment was considered.

```bash
gh api repos/[owner]/[repo]/pulls/[PR_NUMBER]/comments/[COMMENT_ID]/replies \
  -f body="Fixed in [commit sha] — [one line on what changed]."
```

Keep replies factual and one or two lines: what was done, or why not. Resolve threads you fixed if the
repo's workflow expects it (`resolveReviewThread` via `gh api graphql`); leave **needs-human** threads
open.

## Step 7 — Report

Return to the caller:

```markdown
## PR

[PR URL] — [branch] → [base], [N] commits

## Automated review

Reviewer: [matched author login, or "none responded" / "not configured"] · [N] findings

| # | File:line | Finding | Disposition |
|---|-----------|---------|-------------|
| 1 | `src/a.ts:42` | [one-line summary] | ✅ Fixed in `abc1234` |
| 2 | `src/b.ts:10` | [one-line summary] | ⛔ Rejected — [reason] |
| 3 | `src/c.ts:88` | [one-line summary] | 🙋 Needs human — [what decision is needed] |

## Gates after fixes

[table, or "no fixes applied — gates unchanged"]

## Outstanding

[anything unresolved, or "none"]
```

Then, if this was run interactively rather than as a subagent, show the same table to the user and
stop for their review before Step 9 (`gsdl-close-loop`).

---

## Edge Cases

### No review bot is installed on the repo
No bot account will ever comment. After one poll cycle with no bot check registered, report "no
automated review bot is configured on [repo]" and return — don't burn the full 10 minutes. Suggest
setting `Forge → review-bot: none` in the config so future runs skip the wait.

### The bot posts more findings after your fixes
Expected — it re-reviews new commits. Poll once more after pushing fixes (same 10-minute cap) and run
the triage loop again. Cap at **two** fix rounds; if findings keep arriving after that, escalate to
the caller rather than looping.

### Findings conflict with SPEC.md
`SPEC.md` wins — it's the reviewed, agreed plan. Class it **needs-human** and quote both the finding
and the SPEC line it conflicts with.

### PR already merged
Report it and stop. Do not push to a merged branch.

### Repo is not on GitHub
Return a BLOCKER telling the caller to set `Forge → provider: none` (the pipeline then goes straight
to Step 9) or to open the PR manually and pass its URL back.

---

## Rules

1. **Never open a PR on failing gates** — Step 7 must be green first
2. **Never open a PR from the default branch**, and never open a second PR for a project
3. **"No response" is not "no findings"** — report the difference explicitly
4. **Every finding gets a disposition** — fixed, rejected with a reason, escalated, or out-of-scope;
   never silently ignored
5. **Only bot comments are triaged here** — a human reviewer's comments are for the human review that
   follows, not for this step to answer
6. **Re-run gates after applying fixes** — and record the result in `verify-report.md`
7. **Fixes stay scoped to the finding** — no drive-by refactors in a bot-fix commit
8. **Escalate rather than decide** anything that changes agreed scope or needs a product call
9. **Two fix rounds maximum** before escalating
