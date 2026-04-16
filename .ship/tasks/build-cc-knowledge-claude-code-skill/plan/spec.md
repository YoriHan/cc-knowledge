# Spec: cc-knowledge Claude Code Skill

## Overview

`cc-knowledge` is a portable, installable Claude Code skill that turns any user's `~/.claude/projects/` conversation history into a structured personal knowledge vault. It ships as a GitHub-installable skill repository with an interactive setup script, a background ingest daemon, scaffold templates, and a `/o` slash command.

The project generalizes the existing Chinese-language `daily_ingest.py` and `o.md` slash command into a language-agnostic, multi-user installable package.

---

## Deliverables

### 1. `SKILL.md` — `/o` slash command definition

Defines the `/o` command that Claude Code users invoke manually after a session to summarize and file what was learned.

**Behavior:**

1. Read `_system/index.md` from the configured vault to understand current structure and existing pages.
2. Create a new session recap file at `session-recaps/YYYY-MM-DD Title.md` with the following sections:
   - Conversation topic (one-line summary)
   - What was done (bullet list)
   - Methodology / approach used
   - ✅ What went well
   - ⚠️ Improvements / what could be better
   - 💡 Principles extracted (2–4, cross-project reusable)
   - 🔧 Engineering insights (tools/tech discoveries)
   - 🙋 Q&As (questions the user asked, in plain-language format)
3. Update concept-library pages:
   - If concept already exists in index → append to its "session log" section
   - If concept is new → create `concept-library/<Concept Name>.md`
4. Sync Q&As to the appropriate `qa-handbook/` file:
   - Claude Code operations → `qa-handbook/claude-code-ops.md`
   - Git/GitHub → `qa-handbook/git-github.md`
   - General/other → `qa-handbook/general.md`
5. Update `_system/index.md` — add new recap to the session-recaps table.
6. Append to `_system/log.md` — record operation, trigger (`/o`), topic, files created/updated.

**Constraints:**
- Vault path is read from MCP filesystem server config (or env); not hardcoded.
- No fabricated documentation links; only search suggestions.
- Language of output matches the user's configured language (set during setup).
- Concept library: update first, create only if genuinely new.

---

### 2. `setup` — interactive installer (bash, executable)

An interactive bash script run once to configure the vault.

**Questions asked (3):**

| # | Question | Default |
|---|----------|---------|
| 1 | Language for your vault? (e.g. English, 中文, Español) | English |
| 2 | Vault path? | `~/Documents/CC-Knowledge` |
| 3 | Anthropic API key? | (required, no default) |

**Actions performed:**

1. Create vault directory structure by copying everything from `scaffold/`.
2. If language != English: call Claude API (`claude-sonnet-4-6`) to translate:
   - All scaffold template filenames that contain natural-language words (e.g. `mistake-log.md` → `记录失误.md`)
   - Content of `_system/index.md` (section headers, table headers, descriptive text)
3. Write `~/.cc-knowledge/config.json`:
   ```json
   {
     "vault_path": "/absolute/path/to/vault",
     "language": "English",
     "api_key": "sk-ant-..."
   }
   ```
4. Configure MCP filesystem server in `~/.claude.json`:
   - Add entry under `mcpServers` key pointing filesystem server at vault path.
   - Must not clobber existing MCP entries.
5. Copy `bin/daily_ingest.py` into vault at `_system/bin/daily_ingest.py` with correct paths substituted (vault path, config path).
6. Create launchd plist from `launchd/template.plist`, substituting:
   - `__PYTHON__` → `$(which python3)`
   - `__SCRIPT_PATH__` → vault `_system/bin/daily_ingest.py`
   - `__API_KEY__` → Anthropic API key
7. Install plist at `~/Library/LaunchAgents/com.user.cc-knowledge.plist` and run `launchctl load`.
8. Print success summary with vault path, plist status, and first-run instructions.

**Error handling:**
- Abort with clear message if `python3` not found or `anthropic` package missing.
- Warn (do not abort) if launchctl load fails (user can load manually).
- macOS-only notice: launchd step is skipped silently on Linux/WSL with a printed note.

---

### 3. `bin/daily_ingest.py` — background auto-ingest daemon

Generalized from `~/Documents/scripts/daily_ingest/daily_ingest.py`. All Chinese-specific strings replaced with language-agnostic equivalents; language read from config.

**Config source:** `~/.cc-knowledge/config.json`

Fields used:
- `vault_path` — where to write files
- `api_key` — Anthropic API key (also sets `ANTHROPIC_API_KEY` env var)
- `language` — output language for generated content

**Input:** Scans `~/.claude/projects/` recursively for all `*.jsonl` files (not just one subdirectory).

**State tracking:** `~/.cc-knowledge/ingest_state.json`
- Keyed by file path (relative to `~/.claude/projects/`)
- Stores: `processed_at`, `skipped`, `reason`, `date`, `title`, `files`

**Processing pipeline:**

Pre-checks (skip if any fail):
- File modified more than 60 minutes ago (cooldown — conversation likely over)
- At least 4 user messages in conversation

Phase 1 — Decision (`claude-sonnet-4-6`, max 512 tokens):
- Input: first 8000 chars of formatted conversation
- Output JSON: `should_document`, `skip_reason`, `date`, `title`, `has_qa`, `qa_topics`, `new_concepts`, `has_mistake`
- If `should_document=false` → mark skipped, continue

Phase 2a — Recap generation (`claude-sonnet-4-6`, max 3000 tokens):
- Full conversation (up to 25000 chars)
- Output: Markdown session recap in user's language
- Written to `session-recaps/YYYY-MM-DD Title.md` (skip if file already exists)

Phase 2b — Q&A extraction (`claude-haiku-4-5-20251001`, max 1000 tokens):
- Only runs if `has_qa=true` in decision
- Runs once per active `qa_topics` category
- Appends to appropriate `qa-handbook/` file

Post-processing:
- Update `_system/index.md` (insert new recap row before a stable marker)
- Append to `_system/log.md`
- Save updated state

**CLI flags:**
- `--dry-run` — print actions, write nothing
- `--force <uuid>` — reprocess one specific file regardless of state
- `--all` — reprocess all files

**Log file:** `~/.cc-knowledge/ingest.log` (timestamped lines, appended)

---

### 4. `scaffold/` — English template files

All templates are English. Setup translates them if needed.

| Path | Description |
|------|-------------|
| `_system/index.md` | Table of contents. Has sections: Session Recaps (table), Concept Library (table), QA Handbook, Methodology, Engineering Insights, Projects, Open Source Library, Weekly Digest, Mistake Log |
| `_system/log.md` | Append-only operation log. Has header and one example entry. |
| `_system/SCHEMA.md` | Working manual: explains folder purposes, file naming conventions, how /o works, how daily_ingest works |
| `concept-library/.gitkeep` | Placeholder |
| `session-recaps/.gitkeep` | Placeholder |
| `methodology/.gitkeep` | Placeholder |
| `engineering-insights/.gitkeep` | Placeholder |
| `mistake-log/mistake-log.md` | Template with category headers: 🔧 Operations, 🧭 Direction, 💬 Communication, 🤖 AI errors. Includes stats table. |
| `projects/.gitkeep` | Placeholder |
| `open-source-library/README.md` | Index table with columns: Name, Category, Description, Added |
| `weekly-digest/.gitkeep` | Placeholder |
| `qa-handbook/claude-code-ops.md` | Template with header and instructions |
| `qa-handbook/git-github.md` | Template with header and instructions |
| `qa-handbook/general.md` | Template with header and instructions |

---

### 5. `launchd/template.plist`

macOS LaunchAgent plist that runs `daily_ingest.py` daily.

Placeholders:
- `__PYTHON__` — absolute path to python3
- `__SCRIPT_PATH__` — absolute path to installed daily_ingest.py
- `__API_KEY__` — Anthropic API key (set as environment variable)

Schedule: once per day (StartCalendarInterval, Hour: 9, Minute: 0) or on load.

Standard output/error redirected to `~/.cc-knowledge/ingest.log`.

---

### 6. `README.md`

Contents:
- What cc-knowledge is (one paragraph)
- macOS-only notice (launchd; Linux users must set up cron manually)
- Prerequisites: Python 3.10+, `anthropic` pip package, Claude Code
- Installation steps (clone repo, run `./setup`)
- What you get (vault structure overview)
- How to use `/o` command
- Manual ingest commands (`--dry-run`, `--force`, `--all`)
- Config file location

---

## Non-Goals

- Windows support (not planned)
- Multi-vault support (single vault per user)
- GUI or web interface
- Obsidian plugin (may be future work)

---

## Key Design Decisions

1. **Language translation happens at install time**, not at runtime. Once the vault is set up in a language, all subsequent writes use that language (prompts instruct models to output in configured language).

2. **`daily_ingest.py` reads config from `~/.cc-knowledge/config.json`**, not environment variables, so it works reliably under launchd (which has no user shell environment).

3. **Concept library uses update-first semantics** — the `/o` command must check `_system/index.md` before creating any new concept page.

4. **MCP filesystem server** gives Claude Code access to read vault files during `/o` execution, enabling it to actually open and update concept library pages.

5. **State file keyed by relative path** (not filename) to handle multiple projects with same-named JSONL files.
