---
name: journal-organize
description: >-
  Organize a daily journal's Notes section by grouping flat bullet points
  into themed sub-sections with ### topic headings. Preserves all original
  content — only adds structure. Use when asked to organize a journal,
  clean up today's notes, group my notes, or tidy the daily.
compatibility: Requires Obsidian MCP server
metadata:
  author: pshickeydev
  version: "1.0"
---

## Critical Rule — Never overwrite existing journals

**NEVER use `write_note` on a journal file that already exists.** User-written notes, meeting records, and freeform content accumulate in journals throughout the day. Using `write_note` (mode `overwrite`) would destroy that content. Only use `patch_note` to modify existing journals.

**Section ownership:** This skill owns the `## Notes:` section only. Do NOT modify the Tasks/Targets section — that is owned by task skills (`task-create`, `task-daily`, `task-status`, `task-sync-jira`).

## Procedure

### Step 1 — Determine date

If the input arguments contain a date in `YYYY-MM-DD` format, use that date. Otherwise use the **system date** from your environment context (the `Today's date` field in the system prompt). Do NOT derive the date from Obsidian note content, vault metadata, or any other source.

### Step 2 — Read the journal

Read `journals/{date}.md` using the Obsidian `read_note` tool.

**If the journal does not exist**, inform the user and stop. This skill does not create journals — use `task-daily` or `journal-note` to create one first.

**If the journal exists**, proceed to Step 3.

### Step 3 — Check if already organized

Examine the `## Notes:` section. If it already contains `###` sub-headings, inform the user:

```
Journal {date} is already organized ({N} topic sections found).
```

Ask whether the user wants to re-organize (which will replace existing topic groupings) or cancel. If cancel, stop.

### Step 4 — Extract notes content

Isolate everything in the `## Notes:` section:
1. Find the `## Notes:` header.
2. Collect all content from that header to the end of the file (the Notes section is always last).
3. Parse individual bullet points. Each top-level `- ` line is one note entry. Continuation lines (indented or not starting with `- `) belong to the preceding bullet.

If the Notes section is empty or contains only the template placeholder (`- `), inform the user there is nothing to organize and stop.

### Step 5 — Group by topic

Analyze each bullet point and assign it to a topic group. Grouping rules:

1. **Identify themes** from the content of each bullet. Look for:
   - Project/repo names (e.g. `progress-tracker`, `ai-security-harness`)
   - Jira keys (e.g. `ACM-28557`, `HCMSEC-3038`)
   - Activity types (e.g. meeting notes, reading/articles, tooling)
   - Related work streams (bullets about the same system or task)

2. **Choose concise heading names.** Follow patterns from existing organized journals:
   - Project-scoped: `### progress-tracker MR !22`
   - Activity-scoped: `### Meeting: chickenwing sync`
   - Tool/project: `### container-sha2tag`
   - Reading: `### Reading: {article title}`
   - Catch-all: `### Misc` (only for genuinely unrelated single bullets)

3. **Preserve bullet order** within each group. Do not reorder bullets — only move them into groups.

4. **Single-bullet groups are fine.** Not every topic needs multiple bullets. A single bullet about a distinct topic gets its own heading.

### Step 6 — Present the plan

Show the user the proposed organization **without modifying the journal yet**:

```
## Proposed organization for {date}:

### {Topic 1} ({N} bullets)
- {first few words of bullet 1}...
- {first few words of bullet 2}...

### {Topic 2} ({N} bullets)
- {first few words of bullet 1}...

Apply this organization?
```

Truncate each bullet preview to ~80 characters for readability.

Wait for user confirmation before proceeding. If the user requests changes to the grouping (e.g. merge two topics, rename a heading, move a bullet), adjust the plan and re-present.

### Step 7 — Apply the organization

Use the Obsidian `patch_note` tool to replace the Notes section content.

**Build the replacement content:**
1. Keep the `## Notes:` header.
2. For each topic group, emit:
   - A blank line before the heading (for readability).
   - `### {Topic Heading}`
   - Each bullet point from that group, exactly as it appeared in the original (preserving full text, links, formatting).
3. End with a trailing newline.

**Apply the patch:**
- `oldString`: the entire original Notes section content (from `## Notes:` to end of file).
- `newString`: the reorganized Notes section content.

### Step 8 — Confirm

Read the journal back using the Obsidian `read_note` tool to verify the patch applied correctly.

Print:
```
Organized {date} journal: {N} notes grouped into {M} topics.
Topics: {comma-separated list of heading names}
```

## Gotchas

- **NEVER use `write_note` on an existing journal.** Always use `patch_note`. This is critical — journals accumulate content throughout the day.
- **Preserve bullet content exactly.** This skill only adds `###` headings and regroups — it does NOT rewrite, summarize, expand, or editorialize bullet text. The user's words are preserved verbatim.
- **Do NOT touch the Tasks/Targets section.** Only modify content under `## Notes:`.
- **Handle multi-line bullets carefully.** A note entry may span multiple lines (e.g. a bullet followed by indented sub-bullets or continuation text). Keep the full entry together when moving it into a topic group.
- **The Notes section is always last** in the current journal format. There is no content after it. The `oldString` for the patch should match from `## Notes:` through the end of the file content.
- `patch_note` cannot replace with empty string — use a single space if needed.
- Do NOT use `search_notes` for finding journals — use `read_note` with the exact path `journals/{date}.md`.
- If the user has `###` headings already but wants to re-organize, the `oldString` must include the existing headings and all content under them.
