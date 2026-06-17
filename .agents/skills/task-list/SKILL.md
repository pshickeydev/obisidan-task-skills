---
name: task-list
description: >-
  Show a dashboard of tasks from the Obsidian vault. Filters by status
  (open, doing, done, cancelled, active), priority (highest, high, medium,
  low), or tag (#security, #konflux). Default: show open and doing tasks.
  Use when asked to list tasks, show the board, see what's in progress,
  or check task status.
compatibility: Requires Obsidian MCP server
metadata:
  author: pshickeydev
  version: "1.1"
---

## Procedure

### Step 1 — Parse filters

The input arguments may contain zero or more filters separated by spaces. Recognized filters:

| Filter type | Keywords |
|-------------|----------|
| Status | `open`, `doing`, `done`, `cancelled`, `active` (shorthand for open + doing) |
| Priority | `highest`, `high`, `medium`, `low` |
| Tag | Any word starting with `#` (e.g. `#security`, `#konflux`) |

- If no arguments or `all`: default to `active` (open + doing).
- Multiple filters are ANDed together (e.g. `open high` means open tasks with high priority).

### Step 2 — Read task metadata

1. List all files in `tasks/` using the Obsidian `list_directory` tool.
2. Read all frontmatter with the Obsidian `read_multiple_notes` tool (`includeFrontmatter: true`, `includeContent: false`) in batches of 10.
3. Filter in Step 3 after all frontmatter is loaded.

### Step 3 — Filter and sort

Apply the parsed filters against each task's frontmatter:
- **Status filter**: match `status` field (for `active`, match both `open` and `doing`)
- **Priority filter**: match `priority` field
- **Tag filter**: check if the tag (without `#` prefix) appears in the `tags` array

Handle missing frontmatter gracefully:
- Missing `status`: treat as `open`
- Missing `priority`: treat as `medium`
- Missing `tags`: treat as empty array

Sort results:
1. Priority descending: highest → high → medium → low → empty
2. Within same priority: `created_date` ascending (oldest first)

For the `done` filter: limit to tasks completed in the last 30 days (by `completed_date`) unless the user's input includes `all` (e.g. `done all`).

### Step 4 — Render the dashboard

Present results as a markdown table:

```
## Tasks: {filter description}

| Task | Status | Priority | Created | Jira |
|------|--------|----------|---------|------|
| {title} | {status} | {priority} | {created_date} | {jira key or —} |

**Total: {N} tasks** ({X} open, {Y} doing, ...)
```

For the Jira column:
- If `jira_link` is populated, extract the Jira key from the URL (last path segment) and display it as a link.
- If empty, show `—`.

If no tasks match the filter, say so explicitly.

### Step 5 — Suggest follow-ups

After the table, suggest relevant next actions:
- `/task-status {slug} doing` — to start working on a task
- `/task-status {slug} done` — to mark one complete
- `/task-create` — to add a new task

## Gotchas

- Do NOT use `get_notes_info` for filtering — it returns file metadata only (size, modified date), not frontmatter field values.
- Do NOT use `search_notes` with `searchFrontmatter: true` to filter by status or priority — it does fuzzy text matching, not exact field matching, so `status: open` will also match tasks with `status: done` (the word "open" appears elsewhere) and return false positives. Always do a full batch read and filter in-memory.
- Always pass `includeContent: false` to `read_multiple_notes` when you only need frontmatter. This avoids loading note bodies into context.
- Some tasks have empty string `""` for priority. Treat these the same as `medium` for sorting but display as `—` in the table.
- The `done` filter without `all` limits to 30 days to keep output manageable. Mention this limit in the output so the user knows they can add `all` to see everything.
