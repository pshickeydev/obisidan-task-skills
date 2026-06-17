---
name: task-status
description: >-
  Update a task's status in the Obsidian vault. Transitions between open,
  doing, done, and cancelled. Sets completed_date or cancelled_date
  automatically and updates today's journal checkbox. Use when asked to
  start, finish, complete, close, cancel, or reopen a task.
compatibility: Requires Obsidian MCP server
metadata:
  author: pshickeydev
  version: "1.2"
---

## Procedure

### Step 1 — Parse input

The input arguments should be `<task-slug> <status>`.

- If both slug and status are provided, proceed to Step 2.
- If only a slug is provided (no recognized status keyword), ask the user for the target status. Options: `doing`, `done`, `cancelled`, `open`.
- If no arguments at all, list all files in `tasks/` using the Obsidian `list_directory` tool, then read frontmatter with the Obsidian `read_multiple_notes` tool (`includeFrontmatter: true`, `includeContent: false`) in batches of 10. Filter for tasks with `status: "doing"`. Present the results and ask which task to update.

Normalize the slug: strip `.md` extension if present.

### Step 2 — Read the task

Use the Obsidian `read_note` tool on `tasks/{slug}.md`.

If the file does not exist, search for partial matches using the Obsidian `search_notes` tool with the slug as query. Present any matches and ask the user to confirm.

Extract the current `status` from frontmatter.

### Step 3 — Validate the transition

| Current status | Allowed transitions |
|----------------|---------------------|
| open | doing, done, cancelled |
| doing | done, open, cancelled |
| done | open (reopen) |
| cancelled | open (reopen) |

If the target status equals the current status, warn the user and take no action.

If the transition is not in the table above, warn the user and suggest valid options.

### Step 4 — Update frontmatter

Use the Obsidian `update_frontmatter` tool on `tasks/{slug}.md` with `merge: true`:

| Target status | Fields to set |
|---------------|---------------|
| doing | `status: "doing"` |
| done | `status: "done"`, `completed_date: "{today YYYY-MM-DD}"` |
| cancelled | `status: "cancelled"`, `cancelled_date: "{today YYYY-MM-DD}"` |
| open (from done) | `status: "open"`, `completed_date: ""` |
| open (from cancelled) | `status: "open"`, `cancelled_date: ""` |

### Step 5 — Update today's journal

Read today's journal at `journals/{today YYYY-MM-DD}.md` using the Obsidian `read_note` tool.

If the journal exists and contains `- [ ] [[{slug}]]`:
- For transitions to `done` or `cancelled`: use the Obsidian `patch_note` tool to change `- [ ] [[{slug}]]` to `- [x] [[{slug}]]`.
- For transitions to `doing` or `open`: leave the checkbox as `- [ ]` (unchecked).

If the journal exists but does not contain the task slug, and the transition is to `done`:
- Add `- [x] [[{slug}]]` to the Tasks section using `patch_note`, after the last existing checkbox line. Do NOT write to the Notes section — that is owned by the `journal-note` skill.

If the journal does not exist, skip journal updates.

### Step 6 — Report

Print:
- Task title (from frontmatter)
- Status transition: `{old status}` → `{new status}`
- Date fields updated (if any)
- Journal update confirmation (if applicable)

## Gotchas

- The slug may be provided as a partial match (e.g. `proj-123` instead of `proj-123-fix-login-timeout`). If the exact file is not found, list the `tasks/` directory and match by prefix.
- Do NOT use `search_notes` with `searchFrontmatter: true` to filter by status — it does fuzzy text matching and returns false positives. Always batch-read frontmatter and filter in-memory.
- Only update today's journal. Never modify past journal entries.
- When reopening a task, clear only the relevant date field (`completed_date` or `cancelled_date`), not both.
