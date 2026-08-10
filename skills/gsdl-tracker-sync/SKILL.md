---
name: gsdl-tracker-sync
description: "Perform issue-tracker writes for GSDL against whichever tracker is configured — Linear, Jira, GitHub Issues, Notion, or none (local markdown). Creates sub-items under a parent work item, comments on items, and transitions item status. Use when a GSDL step needs to push something to the tracker: sub-item creation at decompose time, slice completion at execute time, or item closure at close-loop time. Mechanical API work only: it never decides what to write, only writes what it is given. Runs on the SMALL tier."
tier: small
disable-model-invocation: true
---

# Tracker Sync

GSDL's tracker write path. Takes an already-decided payload — slice titles, a comment body, a target
state — and applies it to whichever tracker the project is configured for. This skill makes **no
judgment calls**: it does not author comments, choose slice boundaries, or pick which items to close.
Callers decide; this skill executes.

## Tier

**SMALL.** Every operation here is: resolve an ID, POST a mutation, report the result. There is no
reasoning to spend a larger model on, and tracker round-trips are frequent enough across the
pipeline that the cost difference is real. The orchestrator spawns a SMALL subagent for each of these.

**Non-interactive.** Never ask the user anything. If credentials are missing, an ID won't resolve, or
a mutation fails, return a **BLOCKER** describing exactly what failed and what the caller should do —
including the exact text a human could paste into the tracker manually.

## Provider Resolution

Read `.planning/gsdl.config.md` (and any per-project override) per the `gsdl` skill's `config.md`.
The `## Tracker` section decides everything:

| `provider` | Sub-item is | Parenting | Status is |
|---|---|---|---|
| `linear` | An issue with `parentId` | native sub-issue | a workflow state on the team |
| `jira` | A `Sub-task` issue | `fields.parent` | a transition on the issue's workflow |
| `github` | An issue in the same repo | task-list entry in the parent issue body | open / closed (+ optional status label) |
| `notion` | A page in the tracker database | a relation property to the parent page | a `Status`/`Select` property value |
| `none` | A line in `tasks.md` | its parent slice | the checkbox, plus a `worklog.md` entry |

Use the `access` method recorded in the config (`mcp` or `api`). If none is recorded, try MCP first,
then the credential path, and **write back whichever worked**. Use the same method for every call in
one invocation.

Whichever method you use, the constraints below are identical — MCP is not a relaxed path.

---

## Operation A — Create Sub-Items (called from `gsdl-decompose`, Step 4)

### Inputs

- `PARENT_ITEM_ID` — the item from `seed.md`, or `none`
- `PROJECT_ID` — Linear team ID / Jira project key / `owner/repo` / Notion database ID
- `SLICES` — an ordered list of `{ title, description }`, already written by the caller

### Invariants for every provider

1. **Apply the configured labels** — `## Tracker` → `labels` (default `gsdl`), so agent-generated
   items stay filterable. Resolve the label; create it once if the provider requires labels to exist
   first, then reuse it.
2. **Parent every sub-item** to `PARENT_ITEM_ID` when one was given. With `none`, create standalone
   items only if the caller explicitly said to — otherwise return a BLOCKER.
3. **Never assign a sprint, cycle, or iteration.** The parent item is the planning unit; its slices
   are not. Only pass one if the caller explicitly supplied it for a slice.
4. **Never invent a status.** New sub-items land in the project's default/backlog state.
5. **Idempotency** — if the caller passes an existing identifier for a slice, update that item
   instead of creating a duplicate.

### Linear

**MCP**: `save_issue` (or `create_issue`) per slice with `title`, `description`, `parentId`,
`teamId`, and `labelIds`. Omit `cycleId`.

**API**:

```bash
# resolve a label (create it on the team once if absent)
curl -s -X POST -H "Content-Type: application/json" -H "Authorization: $LINEAR_API_KEY" \
  -d '{"query":"{ issueLabels(filter: { name: { eq: \"gsdl\" }, team: { id: { eq: \"TEAM_ID\" } } }) { nodes { id name team { id } } } }"}' \
  https://api.linear.app/graphql

# create the sub-issue
curl -s -X POST -H "Content-Type: application/json" -H "Authorization: $LINEAR_API_KEY" \
  -d '{
    "query": "mutation($input: IssueCreateInput!) { issueCreate(input: $input) { success issue { id identifier url } } }",
    "variables": { "input": {
      "teamId": "TEAM_ID", "parentId": "PARENT_ITEM_ID",
      "title": "SLICE_TITLE", "description": "SLICE_DESCRIPTION_MARKDOWN",
      "labelIds": ["LABEL_ID"]
    } }
  }' https://api.linear.app/graphql
```

If a configured label name resolves to a **child of a label group**, use the child's ID — the group
itself is not assignable. If a configured label doesn't exist and can't be created at team level,
return a BLOCKER rather than creating a stray workspace-level label.

### Jira

**API** (`$JIRA_BASE_URL`, `$JIRA_EMAIL`, `$JIRA_API_TOKEN`):

```bash
curl -s -X POST -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  -d '{"fields":{
        "project":{"key":"PROJ"},
        "parent":{"key":"PROJ-118"},
        "issuetype":{"name":"Sub-task"},
        "summary":"SLICE_TITLE",
        "description":{"type":"doc","version":1,"content":[ /* ADF */ ]},
        "labels":["gsdl"]
      }}' \
  "$JIRA_BASE_URL/rest/api/3/issue"
```

Jira descriptions are Atlassian Document Format, not markdown — convert the slice description
(headings, bullets, checkboxes) into ADF blocks. If the project has no `Sub-task` issue type, fall
back to a `Task` linked to the parent with `"Relates"`, and **say so in the return** — don't silently
change the hierarchy.

### GitHub Issues

```bash
gh issue create --repo owner/repo \
  --title "SLICE_TITLE" --body "SLICE_DESCRIPTION_MARKDOWN

Part of #PARENT_NUMBER" \
  --label gsdl
```

GitHub has no universal parent field. Link both ways: reference the parent in the sub-issue body,
and append the new issue to a task list in the parent's body:

```bash
gh issue view PARENT_NUMBER --repo owner/repo --json body
# append "- [ ] #NEW_NUMBER SLICE_TITLE" under a "## Slices" heading, then:
gh issue edit PARENT_NUMBER --repo owner/repo --body-file -
```

Use the repo's sub-issue API instead if it is available on the account — prefer native parenting
when the API supports it, and report which mechanism you used.

### Notion

Create one page per slice in the tracker database, with:
- the title property set to `SLICE_TITLE`
- the slice description as page content blocks
- a relation property pointing at the parent page (whatever the database calls it — resolve from the
  database schema, don't assume a property name)
- the configured labels in the database's multi-select property, if one exists

Return the page URLs. If the database has no relation property to parent with, return a BLOCKER
naming the properties it does have.

### `none`

No API call. Confirm the caller has written the slices to `tasks.md` and return the slice titles with
`identifier: none` so the caller omits tags. Do not create placeholder IDs.

### Return

Each slice's `identifier` and `url`, **in the order given**, so the caller can tag `tasks.md`. On
partial failure, say exactly which slices were created, with their IDs, so the caller can retry the
rest without duplicating.

---

## Operation B — Comment + Transition (called from `gsdl-execute` Step 6 and `gsdl-close-loop` Step 9)

### Inputs

- `ITEM_IDS` — one or more item identifiers
- `COMMENT` — the exact markdown body to post, written by the caller
- `TARGET_STATE` — the state name to move to (e.g. `Done`, `In Review`), or `none` to comment only

**If `TARGET_STATE` is `none`, skip the transition entirely** — post the comment and return. `none`
is not a state name; never match it against real states and never treat it as a resolution failure.

State names vary per team, project, and workflow — **never hardcode an ID**. Resolve `TARGET_STATE`
case-insensitively against the item's real available states. If nothing matches, return a BLOCKER
listing the available names; do not guess a near-match.

### Linear

```bash
# available states
curl -s -X POST -H "Content-Type: application/json" -H "Authorization: $LINEAR_API_KEY" \
  -d '{"query":"{ issue(id: \"ENG-124\") { team { states { nodes { id name type } } } } }"}' \
  https://api.linear.app/graphql

# comment
curl -s -X POST -H "Content-Type: application/json" -H "Authorization: $LINEAR_API_KEY" \
  -d '{"query":"mutation($input: CommentCreateInput!) { commentCreate(input: $input) { success comment { id url } } }","variables":{"input":{"issueId":"ITEM_ID","body":"COMMENT_MARKDOWN"}}}' \
  https://api.linear.app/graphql

# transition
curl -s -X POST -H "Content-Type: application/json" -H "Authorization: $LINEAR_API_KEY" \
  -d '{"query":"mutation($id: String!, $input: IssueUpdateInput!) { issueUpdate(id: $id, input: $input) { success issue { identifier state { name } } } }","variables":{"id":"ITEM_ID","input":{"stateId":"STATE_ID"}}}' \
  https://api.linear.app/graphql
```

MCP equivalents: `list_issue_statuses`, `save_comment`, `save_issue`.

### Jira

```bash
# comment (body is ADF)
curl -s -X POST -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  -d '{"body":{"type":"doc","version":1,"content":[ /* ADF */ ]}}' \
  "$JIRA_BASE_URL/rest/api/3/issue/PROJ-119/comment"

# available transitions, then apply one
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" "$JIRA_BASE_URL/rest/api/3/issue/PROJ-119/transitions"
curl -s -X POST -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  -d '{"transition":{"id":"TRANSITION_ID"}}' \
  "$JIRA_BASE_URL/rest/api/3/issue/PROJ-119/transitions"
```

Jira transitions are named by the **transition**, not the target status. Match `TARGET_STATE`
against each transition's `to.name`.

### GitHub Issues

```bash
gh issue comment 128 --repo owner/repo --body "COMMENT_MARKDOWN"
gh issue close 128 --repo owner/repo --reason completed     # for a done-like TARGET_STATE
```

GitHub issues have only open/closed. Map any done-like state to `close`, and any other state to a
status label (e.g. `status: in review`) **only if that label already exists** — otherwise comment
only and say in the return that the state could not be represented.

### Notion

Comment → append a paragraph block to the page (or use the comments API if available).
Transition → set the `Status`/`Select` property, resolved by name from the database schema.

### `none`

Append to `.planning/[project-name]/worklog.md`:

```markdown
## [YYYY-MM-DD] [Slice N.0 title or item reference]

**State:** [TARGET_STATE, or "comment only"]

[COMMENT body verbatim]

---
```

Create the file if absent. Also ensure the matching checkbox in `tasks.md` reflects a done-like
state. **The record still gets written** — "no tracker" never means "no record".

### Return

Per item: comment URL (or the worklog path), the resulting state name (or
`unchanged (comment-only)`), and any failure verbatim.

---

## Operation C — Read an Item

For callers needing current item data (state, labels, parent, project/team ID, suggested branch):

| Provider | Call |
|---|---|
| Linear | `{ issue(id:"ENG-124") { id identifier title description branchName state { name } team { id key } parent { identifier } labels { nodes { name } } } }` — MCP: `get_issue` |
| Jira | `GET /rest/api/3/issue/PROJ-119?fields=summary,description,status,labels,parent,project` |
| GitHub | `gh issue view 128 --repo owner/repo --json title,body,state,labels,url,number` |
| Notion | `GET /v1/pages/{id}` + `GET /v1/blocks/{id}/children` |
| `none` | Read `tasks.md` and `worklog.md` |

Return the fields verbatim — do not summarize or interpret.

---

## Rules

1. **Never author content** — post the comment body you were given, verbatim. If it's missing,
   that's a BLOCKER, not an invitation to write one.
2. **Never invent a status** — resolve `TARGET_STATE` against the item's real states and report the
   options if it doesn't match. `none` means comment-only: no resolution, no transition, no blocker.
3. **Every method enforces the same invariants** — MCP, API, the local `none` path, and the
   manual-paste blocker all carry the labels and the no-sprint rule. A path that quietly drops one is
   a bug in this skill.
4. **Never move a parent item to a terminal done state** unless the caller explicitly passed one —
   GSDL's default is parent → the configured review state, sub-items → the configured done state.
5. **Never assign a sprint/cycle/iteration** unless the caller passed one.
6. **Report partial failure honestly** — if 3 of 5 sub-items were created, say exactly which 3, with
   their IDs, so the caller can retry the rest without duplicating.
7. **Report provider workarounds** — a Jira `Task`-instead-of-`Sub-task` fallback, a GitHub state
   that could only be a label, a Notion property that didn't exist: say so in the return rather than
   letting the caller assume a clean write.
8. **Write back the working `access` method** to the config the first time you resolve it.
