---
name: task-daily
description: >-
  Create or refresh today's daily journal in the Obsidian vault. Pulls in
  all doing tasks and high-priority open tasks, checks off completions,
  and creates the journal if it doesn't exist. Use when asked to start the
  day, create today's journal, refresh the daily, or check the day's tasks.
compatibility: Requires Obsidian MCP server
metadata:
  author: pshickeydev
  version: "1.3"
---

## Critical Rule — Never overwrite existing journals

**NEVER use `write_note` on a journal file that already exists.** User-written notes, meeting records, and freeform content accumulate in journals throughout the day. Using `write_note` (mode `overwrite`) would destroy that content. Only use `patch_note` to modify existing journals. `write_note` is permitted ONLY when `read_note` confirms the journal does not exist yet.

## Procedure

### Step 1 — Determine date

If the input arguments contain a date in `YYYY-MM-DD` format, use that date. Otherwise use the **system date** from your environment context (the `Today's date` field in the system prompt). Do NOT derive the date from Obsidian note content, vault metadata, or any other source — those may reflect UTC or a different timezone.

If the date is in the past, warn the user that task states reflect current frontmatter, not the state on that day.

### Step 2 — Gather active tasks

1. List all files in `tasks/` using the Obsidian `list_directory` tool.
2. Read all frontmatter with the Obsidian `read_multiple_notes` tool (`includeFrontmatter: true`, `includeContent: false`) in batches of 10.
3. Collect two sets:
   - **Doing tasks**: all tasks with `status: "doing"`
   - **High-priority open tasks**: tasks with `status: "open"` and `priority: "highest"` or `"high"`

### Step 3 — Check for existing journal

Read `journals/{date}.md` using the Obsidian `read_note` tool.

**If the journal does not exist** (read_note returns a not-found error), create it from the vault template:

1. Read `templates/daily.md` using the Obsidian `read_note` tool.
2. Replace the `{{date:YYYY-MM-DD}}` placeholder with the target date.
3. Write the result to `journals/{date}.md` using the Obsidian `write_note` tool. This is the ONLY case where `write_note` is permitted on a journal file.

Then proceed to Step 4 to populate tasks.

**If the journal exists**, proceed directly to Step 4. Do NOT recreate it from the template — the user may have added notes, meeting records, or other content throughout the day.

### Step 4 — Update existing journal

1. **Add missing doing tasks.** For each doing task not already referenced in the journal (check for `[[slug]]` anywhere in content), add `- [ ] [[slug]]` under the task section header.
   - Detect the existing section header: look for `## Tasks:`, `## Targets:`, `Tasks:`, or `Targets:` (with or without the `##` prefix). Preserve whichever form exists.
   - Use the Obsidian `patch_note` tool to insert after the last `- [ ]` or `- [x]` line in that section.
   - If no task section header exists, prepend `Tasks:\n- [ ] [[slug]]` at the top.

2. **Check off completions.** For each task referenced in the journal as `- [ ] [[slug]]`, check if its frontmatter status is `done` and `completed_date` matches the target date. If so, patch `- [ ]` to `- [x]`.

### Step 5 — Present summary

Read the final journal state with the Obsidian `read_note` tool and present:

```
## Daily: {date}

- **In progress:** {N} tasks (list doing task titles)
- **High priority open:** {N} tasks (list titles)
- **Completed today:** {N} tasks (list titles, if any)
```

## Gotchas

- **NEVER use `write_note` on an existing journal.** Journals accumulate user content throughout the day. Always use `patch_note` for modifications. `write_note` is only for initial creation when no journal exists yet.
- Do NOT use `get_notes_info` for filtering — it returns file metadata only, not frontmatter values.
- Do NOT use `search_notes` with `searchFrontmatter: true` to filter by status — it does fuzzy text matching and returns false positives. Always batch-read frontmatter and filter in-memory.
- Always pass `includeContent: false` to `read_multiple_notes` when only frontmatter is needed.
- Preserve existing journal format — don't rewrite `Targets:` to `Tasks:`, don't restructure freeform journals.
- `patch_note` cannot replace with empty string — use a single space if removing a line.
- Only add `doing` and `highest`/`high` open tasks to new journals — don't dump all open tasks.
