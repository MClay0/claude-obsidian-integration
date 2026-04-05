# PRD: Claude Obsidian Integration

## Introduction

A personal knowledge base system that bridges Claude Code and Obsidian. Claude automatically writes structured, interlinked notes to an Obsidian vault as you work — after commits, debugging sessions, key decisions, and discovered patterns. Notes use Obsidian wikilinks to form a navigable graph. The vault also serves as a personal domain (school, scheduling, bills) that Claude can reference for ambient context. Claude accesses the vault via both MCP (for search/query) and direct file reads/writes (for authoring notes).

## Goals

- Claude automatically writes to the vault at meaningful moments (commits, debugging, decisions, patterns, periodically, on-demand)
- Notes are atomic and linked — graph-navigable, not flat logs
- Predefined templates with frontmatter and link slots keep the graph consistent and queryable
- MCP server enables Claude to search and traverse the vault as tools
- Personal domain folder exists with templates for manual use; Claude can reference it contextually
- On-demand logging via natural language ("log this decision") mid-session

## User Stories

### US-001: Initialize vault structure
**Description:** As a user, I want a well-organized vault scaffold so that both Claude and I have a consistent place to write and find notes.

**Acceptance Criteria:**
- [ ] Script creates vault directory at a configurable path (default: `~/vault`)
- [ ] Creates top-level folders: `Projects/`, `Sessions/`, `Decisions/`, `Patterns/`, `Personal/`
- [ ] Creates `Personal/` subfolders: `School/`, `Bills/`, `Scheduling/`
- [ ] Creates `.obsidian/` folder with minimal `app.json` enabling wikilinks
- [ ] Creates a `README.md` at vault root explaining the structure
- [ ] Script is idempotent — safe to re-run without overwriting existing notes
- [ ] Typecheck passes

### US-002: Define note templates
**Description:** As a developer, I want template files for each note type so that Claude always writes consistent, graph-compatible notes.

**Acceptance Criteria:**
- [ ] `templates/project.md` — frontmatter: type, tags, repo, stack, status. Sections: Overview, Key Decisions (link list), Patterns Discovered (link list), Sessions (link list)
- [ ] `templates/session.md` — frontmatter: type, date, project (wikilink), commits. Sections: What happened, Decisions made (link list), Patterns discovered (link list), Loose ends
- [ ] `templates/decision.md` — frontmatter: type, date, project (wikilink), status. Sections: Context, Decision, Tradeoffs, Referenced in (link list)
- [ ] `templates/pattern.md` — frontmatter: type, tags, applies-to. Sections: Description, Code/Example, Gotchas, Discovered in (link list)
- [ ] `templates/personal-note.md` — minimal frontmatter: type, date, tags. Free-form body
- [ ] All templates use `[[wikilink]]` syntax for cross-references
- [ ] Typecheck passes

### US-003: Install and configure Obsidian MCP server
**Description:** As a developer, I want the Obsidian MCP server wired into Claude Code so that Claude can search and query the vault as tools.

**Acceptance Criteria:**
- [ ] Install script adds `mcp-obsidian` (or equivalent) to Claude Code's MCP config at `~/.claude/claude_desktop_config.json`
- [ ] MCP server configured with vault path
- [ ] Claude can call a `search_vault` tool and get back matching note paths and excerpts
- [ ] Claude can call a `read_note` tool to fetch full note content by path
- [ ] MCP config does not break existing MCP entries
- [ ] Install script prints confirmation and restart reminder
- [ ] Typecheck passes

### US-004: Git commit hook — write session note
**Description:** As a user, I want a session note automatically created after every git commit so that a record of what changed and why is always captured.

**Acceptance Criteria:**
- [ ] `post-commit` hook script written to `hooks/post-commit`
- [ ] Install script symlinks or copies hook to `.git/hooks/post-commit` in target repo (configurable)
- [ ] Hook invokes Claude with commit SHA, message, and diff summary as context
- [ ] Claude writes a `Sessions/YYYY-MM-DD-<slug>.md` note using `templates/session.md`
- [ ] Session note includes commit SHA, one-line summary of what changed, and a Loose ends section
- [ ] Session note appends itself to the linked Project note's Sessions list
- [ ] Hook exits 0 (never blocks the commit)
- [ ] Typecheck passes

### US-005: Git commit hook — update project note
**Description:** As a user, I want the relevant Project note kept up to date after each commit so the project overview always reflects current state.

**Acceptance Criteria:**
- [ ] After writing a session note (US-004), hook reads or creates `Projects/<repo-name>.md`
- [ ] If project note doesn't exist, creates it from `templates/project.md` with repo name, detected stack (from package.json / Cargo.toml / etc.), and status: active
- [ ] Appends new session wikilink to the Sessions list in the project note
- [ ] Does not duplicate existing links
- [ ] Typecheck passes

### US-006: On-demand logging via natural language
**Description:** As a user, I want to say "log this decision" or "save this pattern" mid-session and have Claude write the appropriate note immediately.

**Acceptance Criteria:**
- [ ] Claude recognizes natural language triggers: "log this", "save this decision", "log this pattern", "remember this"
- [ ] Claude identifies the correct template based on trigger phrasing (decision → decision.md, pattern → pattern.md, general → session.md)
- [ ] Claude writes note with filled frontmatter and content derived from current conversation context
- [ ] Claude confirms what was written and the file path
- [ ] New note is linked back to the current project note
- [ ] Typecheck passes

### US-007: Periodic mid-session logging
**Description:** As a user, I want Claude to periodically snapshot the session during long conversations so nothing is lost if I don't explicitly ask.

**Acceptance Criteria:**
- [ ] Claude Code hook triggers a vault write every 15 tool calls (configurable)
- [ ] Snapshot appends to the current session note (or creates one if none exists for today)
- [ ] Snapshot includes: current task, recent decisions, any patterns noticed
- [ ] Does not create duplicate session notes for the same day/project
- [ ] Typecheck passes

### US-008: Personal domain templates and Claude reference
**Description:** As a user, I want templates for my personal notes (school, bills, scheduling) and for Claude to be able to reference them when contextually relevant.

**Acceptance Criteria:**
- [ ] `Personal/School/` template: course name, assignment, due date, status, notes
- [ ] `Personal/Bills/` template: bill name, amount, due date, paid status
- [ ] `Personal/Scheduling/` template: date, events list, notes
- [ ] Claude reads personal notes via MCP when context is relevant (e.g. mentions of deadlines, money, scheduling)
- [ ] Claude never writes to `Personal/` — read-only for Claude
- [ ] Typecheck passes

### US-009: CLAUDE.md injection — vault context on session start
**Description:** As a user, I want Claude to automatically load relevant vault context at the start of each session so it has ambient awareness without me having to explain the project.

**Acceptance Criteria:**
- [ ] Hook reads `Projects/<current-repo>.md` at session start and injects it into context
- [ ] If no project note exists yet, hook skips silently
- [ ] Injected context includes: project overview, last 3 session links, recent decisions
- [ ] Hook does not inject more than ~500 lines to avoid context bloat
- [ ] Typecheck passes

## Non-Goals

- No Obsidian plugin development — vault is plain markdown, Obsidian reads it natively
- No sync or cloud storage setup — vault location is local only
- No automatic summarization or AI-generated weekly reviews (v1)
- No tagging UI or search interface beyond what Obsidian provides natively
- No support for non-Claude AI tools in v1
- Claude does not edit or delete existing notes — append and create only

## Technical Considerations

- Vault is plain markdown files on disk — no database, no proprietary format
- MCP server: `mcp-obsidian` (npm) or `obsidian-mcp` — evaluate availability at install time
- Hooks implemented as bash scripts, installed via a `setup.sh` per-repo script
- All Claude writes go through templates — no freeform file creation
- Wikilinks use relative paths within the vault root (Obsidian default behavior)
- Stack detection for project notes: check for `package.json`, `Cargo.toml`, `go.mod`, `requirements.txt`, `*.csproj`
- Session note slugs derived from commit message: lowercase, spaces to hyphens, max 40 chars
