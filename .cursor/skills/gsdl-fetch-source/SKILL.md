---
name: gsdl-fetch-source
description: "Fetch project context from a Linear ticket, Notion doc, or Slite doc URL. Use when the user provides a Linear, Notion, or Slite URL as the starting point for a GSDL project. Extracts title, description, and structured content from the remote source to pre-populate the seed.md file."
disable-model-invocation: true
---

# GSDL Fetch Source

Retrieves content from a remote source (Linear, Notion, or Slite) and formats it into seed.md content for the GSDL pipeline. Called by the `gsdl` orchestrator when a URL argument is detected.

## Inputs

- `SOURCE_URL` — the full URL provided by the user
- `SOURCE_TYPE` (optional) — explicit hint: `linear`, `notion`, or `slite`. If omitted, auto-detect from URL.

## Output

Produces two values for the caller to use:
1. **Suggested project name** — kebab-case, derived from the fetched title
2. **Seed content** — formatted Markdown to write into `seed.md`

---

## Step 1 — Detect Source Type

Examine `SOURCE_URL` to determine the source:

| Pattern | Source |
|---------|--------|
| `linear.app/` | Linear |
| `notion.so/` or `notion.site/` | Notion |
| `slite.com/` | Slite |

If `SOURCE_TYPE` was provided explicitly, use it instead (it may override auto-detection for non-standard URLs).

If the URL does not match any known pattern and no type was given, tell the user the URL is not recognized and ask them to specify the source type explicitly (`linear`, `notion`, or `slite`).

---

## Step 2 — Extract Identifier from URL

### Linear

URL format: `https://linear.app/{team}/issue/{ISSUE-ID}/{slug}`

- Extract `ISSUE-ID` (e.g., `RURU-441`) from the URL path segment after `/issue/`
- Also capture the team slug (e.g., `karbonhq`) for context

### Notion

URL formats:
- `https://www.notion.so/{workspace}/{Page-Title}-{PAGE_ID}`
- `https://www.notion.so/{PAGE_ID}`
- `https://{workspace}.notion.site/{Page-Title}-{PAGE_ID}`

- Extract `PAGE_ID` — the last path segment or the 32-character hex string (with or without hyphens)
- Normalize the ID by removing hyphens if present

### Slite

URL formats:
- `https://slite.com/app/docs/{NOTE_ID}/{title}` (legacy)
- `https://{workspace}.slite.com/p/{NOTE_ID}/{title}`
- `https://slite.com/p/{NOTE_ID}/{title}`

- Extract `NOTE_ID` from the path segment after `/p/` or `/docs/`

---

## Step 3 — Fetch Content

Try each method in order. Stop at the first success.

---

### Method 0 — MCP (highest priority, auth handled automatically)

MCP servers expose source-specific tools directly to the agent with authentication already configured. Check for a relevant MCP server before anything else.

#### Detect configured MCP servers

Read the MCP configuration files to see which servers are active:

```bash
cat ~/.cursor/mcp.json 2>/dev/null
cat .cursor/mcp.json 2>/dev/null
```

Look for server names or tool prefixes that match the source type. Common server names:

| Source | Common MCP server names |
|--------|------------------------|
| Linear | `linear`, `linear-mcp` |
| Notion | `notion`, `notion-mcp` |
| Slite | `slite`, `slite-mcp` |

If a matching server is listed in the config, attempt to use its tools below.

#### Linear MCP

Try the following tools (try each name in order — the exact name depends on how the server is registered):

- `get_issue` with argument `{ "id": "ISSUE-ID" }` (e.g., `RURU-441`)
- `linear_get_issue` with argument `{ "id": "ISSUE-ID" }`

The response typically includes: `title`, `description`, `state`, `priority`, `labels`, `assignee`, `parent`.

If the tool is unavailable (not found / not registered), skip to Method A.

#### Notion MCP

The official Notion MCP server (`@notionhq/notion-mcp-server`) exposes REST-style tools. Try:

- `retrieve-page` or `notion_retrieve_page` with `{ "page_id": "PAGE_ID" }` → returns title and properties
- `retrieve-block-children` or `notion_retrieve_block_children` with `{ "block_id": "PAGE_ID" }` → returns content blocks

Walk the blocks to extract text from `paragraph`, `heading_1/2/3`, `bulleted_list_item`, `numbered_list_item`, `to_do`, and `callout` block types.

If the tools are unavailable, skip to Method A.

#### Slite MCP

If a Slite MCP server is configured, try:

- `get_note` or `slite_get_note` with `{ "id": "NOTE_ID" }` → returns title and content

If unavailable, skip to Method A.

---

### Method A — API key + curl

If no MCP server is configured or available for the source, fall back to direct API calls using environment variable credentials.

#### Linear

1. Check for a Linear API key:
   ```bash
   echo $LINEAR_API_KEY
   ```
2. If found, query the Linear GraphQL API:
   ```bash
   curl -s -X POST \
     -H "Content-Type: application/json" \
     -H "Authorization: $LINEAR_API_KEY" \
     -d '{
       "query": "{ issue(id: \"ISSUE-ID\") { title description priority priorityLabel state { name } labels { nodes { name } } assignee { name } parent { identifier title } } }"
     }' \
     https://api.linear.app/graphql
   ```
   Replace `ISSUE-ID` with the extracted identifier (e.g., `RURU-441`).
3. Parse the JSON response. The content lives at `data.issue`.

#### Notion

1. Check for a Notion integration token:
   ```bash
   echo $NOTION_TOKEN
   ```
2. If found, fetch the page metadata and content:
   ```bash
   # Page metadata (title, properties)
   curl -s \
     -H "Authorization: Bearer $NOTION_TOKEN" \
     -H "Notion-Version: 2022-06-28" \
     https://api.notion.com/v1/pages/{PAGE_ID}

   # Page block content
   curl -s \
     -H "Authorization: Bearer $NOTION_TOKEN" \
     -H "Notion-Version: 2022-06-28" \
     https://api.notion.com/v1/blocks/{PAGE_ID}/children?page_size=100
   ```
3. Extract the title from `properties.title` (or `properties.Name`) in the page response.
4. Extract text content from the blocks response by walking `results[]` and reading `paragraph.rich_text[].plain_text`, `heading_1/2/3.rich_text[].plain_text`, `bulleted_list_item.rich_text[].plain_text`, etc.

#### Slite

1. Check for a Slite API key:
   ```bash
   echo $SLITE_API_KEY
   ```
2. If found, fetch the note:
   ```bash
   curl -s \
     -H "x-slite-api-key: $SLITE_API_KEY" \
     https://api.slite.com/v1/notes/{NOTE_ID}
   ```
3. Parse the response: `title` and `content` (markdown or HTML).

---

### Method B — WebFetch fallback

If neither MCP nor an API key is available (or both fail), attempt a direct WebFetch of `SOURCE_URL`.

- This works for publicly accessible Linear tickets, Notion pages shared with the web, or public Slite docs.
- Use the WebFetch tool with `SOURCE_URL`.
- Extract the meaningful text content from the returned HTML/markdown, ignoring navigation, sidebars, and boilerplate.

---

### Method C — Manual fallback

If all automated methods fail (e.g., private content, no credentials configured):

1. Inform the user which methods were attempted and why they failed.
2. Explain the available options:

   **Option 1 — Configure an MCP server (recommended):**

   | Source | MCP server package | Config docs |
   |--------|-------------------|-------------|
   | Linear | `@linear/mcp` | https://linear.app/docs/mcp |
   | Notion | `@notionhq/notion-mcp-server` | https://developers.notion.com/docs/mcp |
   | Slite | Check Slite docs for MCP support | https://developers.slite.com |

   Add the server to `.cursor/mcp.json` or `~/.cursor/mcp.json`. Once configured, authentication is handled automatically on all future runs.

   **Option 2 — Set an environment variable:**

   | Source | Variable | Where to generate |
   |--------|----------|-------------------|
   | Linear | `LINEAR_API_KEY` | https://linear.app/settings/api |
   | Notion | `NOTION_TOKEN` | https://www.notion.so/my-integrations |
   | Slite | `SLITE_API_KEY` | Slite workspace settings → API |

   **Option 3 — Paste the content manually:**
   Ask the user to copy and paste the document or ticket content directly into the chat. Use that as the seed content.

3. Wait for the user to choose an option before continuing.

---

## Step 4 — Format as Seed Content

Map the fetched data to the seed.md template. Adapt based on what the source provides.

### For Linear tickets:

```markdown
# {issue.title}

## Source

- **Linear ticket**: {ISSUE-ID} — {SOURCE_URL}
- **Status**: {issue.state.name}
- **Priority**: {issue.priorityLabel}
- **Assignee**: {issue.assignee.name or "Unassigned"}
- **Labels**: {comma-separated label names or "None"}
{if parent: "- **Parent issue**: {parent.identifier} — {parent.title}"}

## Initial Idea

{issue.description — use as-is if it is concise; summarize lightly if very long}

## Problem/Opportunity

{Extract from description if present; otherwise derive from title and context}

## Key Features / Acceptance Criteria

{Extract bullet points, checklists, or sub-issues from the description if present}

## Questions/Uncertainties

{Extract open questions from the description, or leave as a placeholder}

## Next Steps

- [ ] Create PRD
- [ ] Generate task list
```

### For Notion docs:

```markdown
# {page title}

## Source

- **Notion doc**: {SOURCE_URL}

## Initial Idea

{First paragraph or summary section of the doc}

## Problem/Opportunity

{Content from any "Problem", "Goal", "Background", or "Overview" section}

## Key Features / Requirements

{Content from any "Requirements", "Scope", "Features", or bullet sections}

## Design Considerations

{Content from any "Design", "UI", "Mockups" section if present}

## Questions/Uncertainties

{Content from any "Open questions", "TBD", or "Unknowns" section}

## Next Steps

- [ ] Create PRD
- [ ] Generate task list
```

### For Slite docs:

```markdown
# {note title}

## Source

- **Slite doc**: {SOURCE_URL}

## Initial Idea

{Opening paragraph or summary}

## Problem/Opportunity

{Content from any goal/problem/background section}

## Key Features / Requirements

{Content from any requirements/features/scope section}

## Questions/Uncertainties

{Content from any open questions section, or placeholder}

## Next Steps

- [ ] Create PRD
- [ ] Generate task list
```

---

## Step 5 — Derive Project Name

Generate a kebab-case project name from the fetched title:

- Strip special characters, keep alphanumerics and spaces
- Replace spaces with hyphens
- Lowercase everything
- Truncate to ≤ 40 characters at a word boundary
- For Linear: prefix with the issue ID for quick recognition (e.g., `ruru-441-rule-calculator-class`)

Present the suggested name to the caller (`gsdl` orchestrator). The orchestrator will confirm it with the user or use it directly.

---

## Step 6 — Return to Caller

Report back to the `gsdl` orchestrator with:

1. **Suggested project name** (kebab-case string)
2. **Seed content** (the formatted Markdown block from Step 4)
3. **Source summary** (one line: what was fetched, from where)

The orchestrator will:
- Use the suggested project name (confirming with the user if needed)
- Write the seed content directly to `.planning/{project-name}/seed.md` during Phase 0 setup (instead of asking the user to describe the idea from scratch)
- Skip the "describe your idea" prompt in Phase 1 since the seed is already populated from the source

---

## Error Handling

| Situation | Action |
|-----------|--------|
| URL matches no known source | Tell user; ask for explicit source type |
| MCP server listed in config but tool call fails | Log the error; fall through to Method A |
| MCP tool not found (server not running / not registered) | Fall through to Method A silently |
| API key missing | Skip Method A; try Method B (WebFetch) |
| API returns error (404, 403) | Report the error; try Method B (WebFetch) |
| WebFetch returns login page / empty content | Fall back to Method C (manual) |
| Fetched content is very short (< 50 chars) | Warn user; ask them to verify the URL and that the doc has content |
