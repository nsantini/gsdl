# GSDL Config — the provider contract

GSDL is not tied to any one issue tracker, documentation platform, code forge, or model
provider. Every skill that touches an external system reads its target from a single config
file, so swapping Linear for Jira (or for nothing at all) is a config edit, not a skill rewrite.

**Canonical location**: `.planning/gsdl.config.md` (workspace-level, shared by all projects).

**Per-project override**: `.planning/[project-name]/gsdl.config.md`. If it exists, its sections
replace the workspace-level ones **section by section** — a project override that only sets
`## Tracker` still inherits `## Docs`, `## Forge`, and `## Models` from the workspace file.

---

## Format

```markdown
# GSDL Config

## Tracker

- provider: linear          # linear | jira | github | notion | none
- access: mcp               # mcp | api | none  (auto-resolved; record what worked)
- project: ENG              # team key / Jira project key / owner-repo / Notion database ID
- labels: gsdl              # comma-separated; applied to every sub-item GSDL creates
- sub-item-done-state: Done         # state a completed slice moves to
- parent-review-state: In Review    # state the parent moves to at close-loop

## Docs

- provider: slite           # slite | notion | confluence | none
- parent: abc123            # parent note / page / space ID or URL

## Forge

- provider: github          # github | none
- review-bot: auto          # auto | none | <bot login, e.g. cursor[bot]>

## Models

- provider: anthropic       # anthropic | openai | google | other
- large:                    # optional explicit override, else the tier default is used
- medium:
- small:
```

Every field is optional. A missing field takes the default below.

---

## Fields

### `## Tracker`

| Field | Values | Default | Meaning |
|---|---|---|---|
| `provider` | `linear` \| `jira` \| `github` \| `notion` \| `none` | `none` | Where work items live |
| `access` | `mcp` \| `api` \| `none` | auto-probed | How GSDL reaches it. Recorded after the first successful call so later runs skip the probe |
| `project` | string | — | Linear team key, Jira project key, `owner/repo` for GitHub Issues, database ID for Notion |
| `labels` | comma-separated | `gsdl` | Applied to every sub-item GSDL creates, so agent-generated items are filterable |
| `sub-item-done-state` | string | `Done` | Resolved by name against the real workflow — never hardcode an ID |
| `parent-review-state` | string | `In Review` | Where the parent item lands at close-loop. GSDL never moves a parent to a terminal state |

**`provider: none` is a first-class mode, not a degraded one.** With no tracker, `tasks.md` *is*
the work-item list, and everything a tracker write would have said is appended to
`.planning/[project-name]/worklog.md` instead. The pipeline's checkpoints are identical.

### `## Docs`

| Field | Values | Default | Meaning |
|---|---|---|---|
| `provider` | `slite` \| `notion` \| `confluence` \| `none` | `none` | Where the decisions document gets published |
| `parent` | ID or URL | — | Parent note (Slite), page (Notion), or space/parent page (Confluence) |

With `none`, the decisions document is written to disk and nothing is published. The document
itself is never optional — only the publish is.

### `## Forge`

| Field | Values | Default | Meaning |
|---|---|---|---|
| `provider` | `github` \| `none` | auto-probed | Where the pull request is opened |
| `review-bot` | `auto` \| `none` \| login | `auto` | `auto` triages findings from whichever review bot commented (BugBot, CodeRabbit, Copilot, Greptile, …); a login pins it to one; `none` skips the wait entirely |

With `provider: none`, Step 8 is skipped and the pipeline goes straight from verify to close-loop.
Skipping is recorded in `progress.md` and in the decisions document — never silently.

### `## Models`

See [`models.md`](./models.md). `provider` selects the tier mapping; `large` / `medium` / `small`
override individual tiers with explicit model IDs when the defaults aren't what you want.

---

## Resolution order

Any skill needing a provider resolves it like this:

1. **Per-project config** — `.planning/[project-name]/gsdl.config.md`, section by section
2. **Workspace config** — `.planning/gsdl.config.md`
3. **Probe** — connected MCP servers first, then credential environment variables:

   | Provider | MCP server names | Env var |
   |---|---|---|
   | Linear | `linear`, `linear-mcp` | `LINEAR_API_KEY` |
   | Jira | `jira`, `atlassian`, `mcp-atlassian` | `JIRA_API_TOKEN` + `JIRA_BASE_URL` + `JIRA_EMAIL` |
   | GitHub Issues | `github`, `github-mcp` | `GH_TOKEN` / `gh auth status` |
   | Notion | `notion`, `notion-mcp` | `NOTION_TOKEN` |
   | Slite | `slite`, `slite-mcp` | `SLITE_API_KEY` |
   | Confluence | `confluence`, `atlassian` | `CONFLUENCE_API_TOKEN` + `CONFLUENCE_BASE_URL` |

4. **Ask the user once**, then **write the answer to the config file** so it is never asked again.

Probing is a fallback, not the normal path. Once a provider is recorded, use it — do not re-probe
on every run, and do not silently switch providers because a probe found something else.

---

## Writing the config

`gsdl-fetch-source` creates `.planning/gsdl.config.md` on the first run in a workspace, filling in
what it could resolve and leaving the rest at defaults. Any skill that resolves a previously
unknown field writes it back.

**Never overwrite a field the user set by hand.** If a probe disagrees with the config, use the
config and tell the user about the mismatch.

---

## Example — Linear + Slite + GitHub

```markdown
# GSDL Config

## Tracker
- provider: linear
- access: mcp
- project: ENG
- labels: gsdl
- sub-item-done-state: Done
- parent-review-state: In Review

## Docs
- provider: slite
- parent: https://myteam.slite.com/p/abc123/Engineering-Decisions

## Forge
- provider: github
- review-bot: auto

## Models
- provider: anthropic
```

## Example — no tracker, local only

```markdown
# GSDL Config

## Tracker
- provider: none

## Docs
- provider: none

## Forge
- provider: none

## Models
- provider: openai
```

This runs the full pipeline against markdown files only: `seed.md` → `SPEC.md` → `tasks.md` →
`verify-report.md` → `decisions-*.md`, with `worklog.md` standing in for tracker comments.
