# AGENTS.md

> This file provides context for AI agents working with this repository.
> It follows the [AGENTS.md convention](https://github.com/anthropics/agent-conventions).

## Project Overview

This repository contains AI agent skills for managing tasks and daily journals in an Obsidian vault via MCP (Model Context Protocol). Skills are plain Markdown files with structured procedures that any compatible agent can follow.

## Repository Structure

```
.agents/skills/          ← Canonical skill definitions (agent-agnostic)
.crush/skills            ← Symlink → .agents/skills (for Crush)
.claude/skills           ← Symlink → .agents/skills (for Claude Code)
CLAUDE.md.example        ← Project context template (copy to CLAUDE.md and customize)
```

### Skill format

Each skill is a directory containing a `SKILL.md` file with:

1. **YAML frontmatter** — `name`, `description` (trigger phrase), `compatibility` (required MCP servers), `metadata`
2. **Procedure section** — numbered steps the agent must follow exactly
3. **Gotchas section** — known pitfalls and edge cases

## MCP Server Dependencies

| MCP Server | Required By | Purpose |
|------------|-------------|---------|
| Obsidian | All skills | Read/write task notes and journals in the vault |
| Jira | task-create, task-sync-jira | Fetch issue data (summary, priority, labels) |

## Key Conventions

### Task frontmatter schema

All task notes require exactly 9 frontmatter fields: `title`, `status`, `priority`, `created_date`, `due_date`, `completed_date`, `cancelled_date`, `jira_link`, `tags`. See `CLAUDE.md` for full details.

### Journal safety rule

**Never use `write_note` on a journal that already exists.** Journals accumulate user content throughout the day. Use `patch_note` for modifications. `write_note` is only permitted when creating a brand-new journal file.

### Obsidian MCP tool limitations

- `search_notes` does fuzzy matching — do not use it to filter by exact frontmatter values
- `read_multiple_notes` accepts max 10 paths per call — batch accordingly
- `patch_note` cannot replace with empty string — use a single space
- Always pass `includeContent: false` to `read_multiple_notes` when only frontmatter is needed

**Reliable pattern:** Batch-read frontmatter via `read_multiple_notes`, then filter in-memory.

## Adding Support for a New Agent

1. Create the agent's dotfile directory (e.g., `.myagent/`)
2. Symlink: `ln -s ../.agents/skills .myagent/skills`
3. Configure the agent to discover `SKILL.md` files from that directory
4. Copy `CLAUDE.md.example` to `CLAUDE.md` and fill in your vault path and Jira domain
