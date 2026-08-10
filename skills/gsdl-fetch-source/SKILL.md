---
name: gsdl-fetch-source
description: "Pick up the starting point for a GSDL project — a tracker work item (Linear, Jira, GitHub Issues, Notion), a specification document (Notion, Slite, Confluence), or a plain idea with no tracker at all — and create .planning/[project-name]/ with seed.md populated from it. Also establishes the workspace provider config on first run. Use when the user gives a ticket URL/ID, a doc URL, or an idea as the starting point for a project, or asks to 'pick up this ticket'. This is Step 1 of the GSDL pipeline. Runs on the SMALL tier."
tier: small
disable-model-invocation: true
---

# Fetch Source

Turns whatever the user is starting from into a project: creates `.planning/[project-name]/` and
writes `seed.md`. This is **Step 1** of the GSDL pipeline:
`Pick up source → Brainstorm → SPEC.md → Slices → Review → Branch → Execute → Verify → PR + bot → Close loop`

GSDL accepts three kinds of starting point, and treats them identically from Step 3 onwards:

| Starting point | Example | Tracker item created later? |
|---|---|---|
| **Tracker work item** | `ENG-441`, `PROJ-45`, `owner/repo#12`, a Linear/Jira/GitHub URL | Yes — slices become sub-items of it |
| **Specification document** | A Notion, Slite, or Confluence URL | Only if a tracker is configured and the user wants a parent item |
| **A plain idea** | "a CLI that syncs local files to S3" | No — `tasks.md` is the work-item list |

**There is no wrong entry point.** A project with no tracker runs the same nine steps and produces
the same artifacts.

## Tier

**SMALL.** This step is mechanical: resolve an ID, call one API, fill a template. All tracker I/O in
GSDL runs on SMALL (see `gsdl-tracker-sync` for the write path) — when the `gsdl` orchestrator runs
this step it spawns a SMALL subagent for it. Invoked standalone on a larger model that's fine, but
there is no judgment here worth paying for.

Running as a subagent means **non-interactive**: don't ask the user anything. If you hit the manual
fallback, an existing project folder, or unparseable input, return a **BLOCKER** to the caller
describing exactly what's needed instead of guessing or overwriting. The orchestrator resolves those
with the user on the LARGE tier.

## Inputs

- `SOURCE` — a work-item URL or ID, a document URL, or the user's plain description of the idea
- `SOURCE_TYPE` (optional) — explicit hint when auto-detection would be wrong

## Output

1. **Project name** — kebab-case, prefixed with the item ID when there is one
2. **`.planning/[project-name]/` folder**, created if absent
3. **`.planning/[project-name]/seed.md`**, populated from the source
4. **`.planning/gsdl.config.md`**, created on the first run in this workspace
5. Returned to the caller: project name, seed path, item ID (or `none`), the tracker's suggested
   branch name (or `none`), and a one-line source summary

---

## Step 1 — Resolve the Provider Config

Read the config contract in the `gsdl` skill's `config.md` (resolve the path the same way sub-skills
are resolved), then read `.planning/gsdl.config.md`.

**If it exists**, use it. Do not re-probe, and do not override a field the user set by hand.

**If it does not exist**, probe once and create it:

1. List connected MCP servers and match them against the provider table in `config.md`.
2. Check the credential environment variables for anything not covered by MCP.
3. Check the forge: `gh auth status` for GitHub.
4. Infer the model provider from the session's own model unless told otherwise.
5. Write `.planning/gsdl.config.md` with what you resolved and defaults for the rest.

Anything genuinely ambiguous — two trackers available, a docs target that needs a parent ID —
is left blank and returned to the caller as a question, **not guessed**. A blank field is resolved
later, on first use, and written back then.

---

## Step 2 — Classify the Source

| Pattern | Source type |
|---|---|
| `linear.app/{team}/issue/{ID}/{slug}` or a bare `ABC-123` matching a Linear team key | Linear item |
| `{site}.atlassian.net/browse/{KEY-123}` or a bare `KEY-123` matching a Jira project key | Jira item |
| `github.com/{owner}/{repo}/issues/{n}`, `{owner}/{repo}#{n}`, or `#{n}` in a GitHub repo | GitHub issue |
| `notion.so/…` / `notion.site/…` | Notion page — an item if it is a row in the configured tracker database, otherwise a document |
| `slite.com/…` or `{workspace}.slite.com/p/…` | Slite document |
| `{site}.atlassian.net/wiki/…` | Confluence document |
| Anything else that is a URL | Unknown — try a generic web fetch, and say so |
| Not a URL or ID | A plain idea — go to Method D |

A **bare ID** is ambiguous between trackers. Disambiguate with the configured tracker provider
first; only if that fails, try the others. `SOURCE_TYPE` overrides all of this.

---

## Step 3 — Fetch the Content

Try each method in order. Stop at the first success. **Use the access method recorded in the config**
if there is one — the ladder below is for resolving it the first time.

### Method 0 — MCP (preferred, auth handled automatically)

| Source | Typical tools |
|---|---|
| Linear | `get_issue` / `linear_get_issue` with `{ "id": "ENG-441" }` |
| Jira | `getJiraIssue` / `jira_get_issue` with the issue key |
| GitHub | `get_issue` / `github_get_issue` with `{ owner, repo, issue_number }` |
| Notion | `retrieve-page` + `retrieve-block-children` with the page ID |
| Slite | `get-note` / `slite_get_note` with the note ID |
| Confluence | `getConfluencePage` / `confluence_get_page` with the page ID |

Walk Notion/Confluence block trees for `paragraph`, `heading_1/2/3`, `bulleted_list_item`,
`numbered_list_item`, `to_do`, and `callout` content.

Record which tool worked so `gsdl-tracker-sync` can reuse the same access path.

### Method A — API + credentials

**Linear** (`$LINEAR_API_KEY`):

```bash
curl -s -X POST -H "Content-Type: application/json" -H "Authorization: $LINEAR_API_KEY" \
  -d '{"query":"{ issue(id: \"ENG-441\") { title description priority priorityLabel branchName state { name } team { id key } labels { nodes { name } } assignee { name } parent { identifier title } } }"}' \
  https://api.linear.app/graphql
```

**Jira** (`$JIRA_BASE_URL`, `$JIRA_EMAIL`, `$JIRA_API_TOKEN`):

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "$JIRA_BASE_URL/rest/api/3/issue/PROJ-45?fields=summary,description,status,priority,labels,assignee,parent,project"
```

Jira descriptions come back as Atlassian Document Format — flatten the `content` tree to markdown
rather than dumping the JSON.

**GitHub Issues** (`gh`, already authenticated):

```bash
gh issue view 12 --repo owner/repo --json title,body,state,labels,assignees,url,number
```

**Notion** (`$NOTION_TOKEN`):

```bash
curl -s -H "Authorization: Bearer $NOTION_TOKEN" -H "Notion-Version: 2022-06-28" \
  https://api.notion.com/v1/pages/{PAGE_ID}
curl -s -H "Authorization: Bearer $NOTION_TOKEN" -H "Notion-Version: 2022-06-28" \
  "https://api.notion.com/v1/blocks/{PAGE_ID}/children?page_size=100"
```

**Slite** (`$SLITE_API_KEY`):

```bash
curl -s -H "x-slite-api-key: $SLITE_API_KEY" https://api.slite.com/v1/notes/{NOTE_ID}
```

**Confluence** (`$CONFLUENCE_BASE_URL`, `$CONFLUENCE_API_TOKEN`, `$JIRA_EMAIL`):

```bash
curl -s -u "$JIRA_EMAIL:$CONFLUENCE_API_TOKEN" \
  "$CONFLUENCE_BASE_URL/wiki/api/v2/pages/{PAGE_ID}?body-format=storage"
```

### Method B — Web fetch fallback

If neither MCP nor credentials are available, fetch the URL directly. Works only for publicly shared
content. Extract the meaningful text, ignoring navigation and boilerplate.

### Method C — Manual fallback

If everything automated fails:

1. Say which methods were attempted and why each failed.
2. Offer: configure the relevant MCP server, set the credential env var, or paste the content
   directly into the chat.
3. Wait for the user's choice.

**As a subagent, do not do this yourself** — return a BLOCKER naming the source, the methods tried,
and the three options, and let the orchestrator run it with the user.

### Method D — No source (a plain idea)

No fetch at all. Use the user's description as the seed content, and mark the project as having no
tracker item. This is a supported path, not a fallback from a failure — do not apologise for it or
push the user towards creating a ticket.

If the description is thin, that is fine: `gsdl-create-spec` (Step 3) asks the clarifying questions.
Capture what the user actually said rather than inventing detail.

---

## Step 4 — Derive the Project Name

- Take the item/document title, or the first clause of the user's description
- Strip special characters (keep alphanumerics and spaces), replace spaces with hyphens, lowercase
- Truncate to ≤ 40 characters at a word boundary
- **Prefix with the item ID when there is one**: `eng-441-rule-calculator-class`,
  `proj-45-billing-webhook`, `gh-12-flaky-retry`. With no item, the title alone.

Confirm the name with the user: *"I picked up ENG-441 — Rule Calculator Class. I'll use
`eng-441-rule-calculator-class` as the project name — does that work?"*

**As a subagent** (non-interactive), skip the prompt — return the derived name and let the caller confirm.

---

## Step 5 — Create the Project Structure

```bash
mkdir -p .planning/[project-name]
```

---

## Step 6 — Write seed.md

```markdown
# {title}

## Source

{one of the Source blocks below}

## Initial Idea

{description — use as-is if concise; summarize lightly if very long}

## Problem/Opportunity

{Extract from the description if present; otherwise derive from the title and context}

## Key Features / Acceptance Criteria

{Extract bullet points, checklists, or sub-items from the description if present}

## Questions/Uncertainties

{Extract open questions from the description, or leave as a placeholder}

## Next Steps

- [ ] Brainstorm / clarify intent
- [ ] Produce SPEC.md
```

### Source blocks

**Tracker item:**

```markdown
- **Tracker**: {linear | jira | github | notion}
- **Item**: {ITEM-ID} — {SOURCE_URL}
- **Status**: {state name}
- **Priority**: {priority label, or "none"}
- **Assignee**: {name, or "Unassigned"}
- **Labels**: {comma-separated, or "None"}
- **Project/Team**: {team key or project key} ({ID})
- **Suggested branch**: {branchName if the tracker exposes one, else "none — derive from the project name"}
{if parent: "- **Parent item**: {identifier} — {title}"}
```

Only Linear exposes a suggested branch name today. For Jira, GitHub, and Notion, write
`none — derive from the project name`; Step 5.5 handles it.

**Document:**

```markdown
- **Document**: {notion | slite | confluence} — {SOURCE_URL}
- **Tracker item**: none (created later if a tracker is configured)
```

**Plain idea:**

```markdown
- **Source**: local idea, captured {YYYY-MM-DD}
- **Tracker item**: none
```

`seed.md` doubles as **Step 2 (Brainstorm / Clarify Intent)**'s starting point. The user may edit it
before `gsdl-create-spec` reads it.

---

## Step 7 — Confirm and Hand Off

Show the user:
- The created `.planning/[project-name]/` folder and `seed.md` path
- A one-line summary of what was picked up
- Whether a provider config was created, and what it resolved to
- That the next step is producing `SPEC.md` via `gsdl-create-spec`

Called from the `gsdl` orchestrator, return the project name, the seed content, the item ID, the
project/team ID, and the suggested branch name, so the orchestrator can write `progress.md` and set
up the working branch at Step 5.5 without re-prompting.

---

## Edge Cases

### Project already exists
If `.planning/[project-name]/` already has a `seed.md`, ask whether to re-fetch and overwrite (the
item may have changed) or keep the existing seed. As a subagent, return a BLOCKER — never overwrite.

### The item has no description
Still create the project; note in `seed.md` that the source had no description and that the
clarifying questions in `gsdl-create-spec` will establish scope from scratch.

### A document that is really a spec
Long documents often already contain goals, requirements, and non-goals. Map them into the matching
`seed.md` sections rather than dumping the whole document into `## Initial Idea`. `SPEC.md` is still
produced at Step 3 — the document is input, not a substitute.

### A bare ID that matches nothing
Report which trackers were tried and ask for a full URL.

## Error Handling

| Situation | Action |
|---|---|
| Input matches no known pattern and is not prose | Ask for a URL, an ID, or a description |
| MCP tool call fails | Log the error; fall through to Method A |
| MCP tool not found | Fall through to Method A silently |
| Credentials missing | Skip Method A; try Method B |
| API returns 404 / 403 | Report the error; try Method B |
| Web fetch returns a login page or empty content | Fall back to Method C |
| Fetched content is very short (< 50 chars) | Warn; ask the user to confirm the source has real content |
| No tracker configured but the input is an item ID | Fetch it anyway if credentials allow, and record `Tracker → provider` in the config |
