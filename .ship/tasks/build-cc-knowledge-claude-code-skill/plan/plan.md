# Plan: cc-knowledge Claude Code Skill

## Stories

---

### Story 1: Project scaffolding and directory layout

**What:** Create the top-level repository structure at `~/Documents/projects/cc-knowledge/` with all directories that will hold the deliverables.

**Directories to create (if not present):**
```
cc-knowledge/
├── bin/
├── scaffold/
│   ├── _system/
│   ├── concept-library/
│   ├── session-recaps/
│   ├── methodology/
│   ├── engineering-insights/
│   ├── mistake-log/
│   ├── projects/
│   ├── open-source-library/
│   ├── weekly-digest/
│   └── qa-handbook/
└── launchd/
```

**Acceptance criteria:**
- All directories exist.
- No files written yet (scaffold content is Story 2).

---

### Story 2: scaffold/ template files

**What:** Write all English-language template files under `scaffold/`.

**Files:**

`scaffold/_system/index.md` — Table of contents with sections:
- Session Recaps: markdown table with columns `File | Title | Date`
- Concept Library: markdown table with columns `Concept | First Seen | Notes`
- QA Handbook: links to the three qa-handbook files
- Other sections (Methodology, Engineering Insights, Projects, Open Source Library, Weekly Digest, Mistake Log) as h2 headers with brief description

`scaffold/_system/log.md` — Append-only log:
```
# Operation Log

<!-- Append new entries at the bottom -->

## [YYYY-MM-DD HH:MM] example-entry | daily_ingest | Example Topic
**Created:** session-recaps/YYYY-MM-DD Example Topic.md
**Source:** abc123.jsonl
```

`scaffold/_system/SCHEMA.md` — Working manual explaining:
- Folder purposes (all 9 folders)
- File naming conventions
- How `/o` command works
- How `daily_ingest.py` works
- How to add entries manually

`scaffold/mistake-log/mistake-log.md` — Template:
```markdown
# Mistake Log

## 🔧 Operations (tool/command mistakes)

## 🧭 Direction (judgment/scope mistakes)

## 💬 Communication (requirements misunderstood)

## 🤖 AI Errors (Claude acting autonomously or incorrectly)

---
## Stats
| Category | Count |
|----------|-------|
| 🔧 Operations | 0 |
| 🧭 Direction | 0 |
| 💬 Communication | 0 |
| 🤖 AI Errors | 0 |
```

`scaffold/open-source-library/README.md` — Index table:
```markdown
# Open Source Library

| Name | Category | Description | Added |
|------|----------|-------------|-------|
```

`scaffold/qa-handbook/claude-code-ops.md`:
```markdown
# QA Handbook: Claude Code Operations

Questions and answers about Claude Code usage, commands, and workflows.

<!-- Entries appended by /o or daily_ingest -->
```

`scaffold/qa-handbook/git-github.md`:
```markdown
# QA Handbook: Git & GitHub

Questions and answers about Git commands and GitHub workflows.

<!-- Entries appended by /o or daily_ingest -->
```

`scaffold/qa-handbook/general.md`:
```markdown
# QA Handbook: General

General questions about AI tools, workflows, and concepts.

<!-- Entries appended by /o or daily_ingest -->
```

All other folders get a `.gitkeep` file.

**Acceptance criteria:**
- All 14 files/placeholders exist under `scaffold/`.
- English only; no hardcoded user paths.

---

### Story 3: `bin/daily_ingest.py`

**What:** Write the generalized auto-ingest script.

**Key differences from the existing `daily_ingest.py`:**
- Config read from `~/.cc-knowledge/config.json` (not hardcoded paths)
- JSONL scan: `Path.home() / ".claude/projects/"` recursively (`rglob("*.jsonl")`) instead of one subdirectory
- State file: `~/.cc-knowledge/ingest_state.json` (not `~/.claude/daily_ingest_state.json`)
- Log file: `~/.cc-knowledge/ingest.log`
- State keyed by relative path from `~/.claude/projects/` root (not just filename)
- All user-facing strings in prompts include instruction to respond in `{config["language"]}`
- `format_conversation` prefix labels are neutral ("User" / "Claude") not Chinese characters
- Truncation message is English ("conversation truncated")
- `update_index` and `update_log` paths use English folder names from scaffold
- `qa_topics` mapping: `"Claude Code Operations"` → `qa-handbook/claude-code-ops.md`, `"Git & GitHub"` → `qa-handbook/git-github.md`, `"General"` → `qa-handbook/general.md`
- Decision system prompt and recap system prompt adapted for multi-language output

**Module structure (same as existing, just generalized):**
```python
load_config() -> dict          # reads ~/.cc-knowledge/config.json
log(msg)                       # to ~/.cc-knowledge/ingest.log
load_state() / save_state()    # ~/.cc-knowledge/ingest_state.json
extract_conversation(path)     # same logic
format_conversation(messages)  # same logic, English labels
call_decision_api(conv_text)   # Phase 1, language-aware prompt
call_recap_api(conv, date, title) # Phase 2a
call_qa_api(conv, category)    # Phase 2b, claude-haiku-4-5-20251001
write_file(rel_path, content)  # relative to vault_path
append_to_file(rel_path, content)
update_index(date, title, recap_path)
update_log(date, title, files, source)
process_file(jsonl_path, state)
main()                         # argparse: --dry-run, --force, --all
```

**Acceptance criteria:**
- Script is importable (no syntax errors).
- Config loading fails gracefully with clear message if `~/.cc-knowledge/config.json` is missing.
- Recursive JSONL scan works across multiple project subdirectories.
- State keys include subdirectory path component to avoid collisions.
- Language string is injected into both Phase 1 and Phase 2 prompts.

---

### Story 4: `launchd/template.plist`

**What:** Write the macOS LaunchAgent plist template.

**Content:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.user.cc-knowledge</string>
  <key>ProgramArguments</key>
  <array>
    <string>__PYTHON__</string>
    <string>__SCRIPT_PATH__</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>ANTHROPIC_API_KEY</key>
    <string>__API_KEY__</string>
  </dict>
  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key>
    <integer>9</integer>
    <key>Minute</key>
    <integer>0</integer>
  </dict>
  <key>StandardOutPath</key>
  <string>__LOG_PATH__</string>
  <key>StandardErrorPath</key>
  <string>__LOG_PATH__</string>
  <key>RunAtLoad</key>
  <false/>
</dict>
</plist>
```

Placeholders: `__PYTHON__`, `__SCRIPT_PATH__`, `__API_KEY__`, `__LOG_PATH__`

**Acceptance criteria:**
- Valid XML/plist syntax.
- All four placeholders present and documented.

---

### Story 5: `setup` bash script

**What:** Write the interactive installer.

**Script outline:**

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Check prerequisites
command -v python3 || { echo "Error: python3 not found"; exit 1; }
python3 -c "import anthropic" 2>/dev/null || { echo "Error: pip install anthropic"; exit 1; }

# 2. Ask 3 questions
read -p "Language [English]: " LANGUAGE
LANGUAGE="${LANGUAGE:-English}"

read -p "Vault path [~/Documents/CC-Knowledge]: " VAULT_PATH
VAULT_PATH="${VAULT_PATH:-$HOME/Documents/CC-Knowledge}"
VAULT_PATH="${VAULT_PATH/#\~/$HOME}"  # expand tilde

read -p "Anthropic API key: " API_KEY
[[ -z "$API_KEY" ]] && { echo "Error: API key required"; exit 1; }

# 3. Copy scaffold
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
cp -r "$SCRIPT_DIR/scaffold/." "$VAULT_PATH/"

# 4. Translate if needed (call Claude API via python3 inline script)
if [[ "$LANGUAGE" != "English" ]]; then
  python3 "$SCRIPT_DIR/bin/translate_scaffold.py" \
    --vault "$VAULT_PATH" --language "$LANGUAGE" --api-key "$API_KEY"
fi

# 5. Write config
mkdir -p "$HOME/.cc-knowledge"
cat > "$HOME/.cc-knowledge/config.json" <<EOF
{
  "vault_path": "$VAULT_PATH",
  "language": "$LANGUAGE",
  "api_key": "$API_KEY"
}
EOF

# 6. Configure MCP in ~/.claude.json
python3 "$SCRIPT_DIR/bin/configure_mcp.py" --vault "$VAULT_PATH"

# 7. Copy daily_ingest.py into vault
mkdir -p "$VAULT_PATH/_system/bin"
cp "$SCRIPT_DIR/bin/daily_ingest.py" "$VAULT_PATH/_system/bin/daily_ingest.py"

# 8. Install launchd plist (macOS only)
if [[ "$(uname)" == "Darwin" ]]; then
  PYTHON_PATH="$(which python3)"
  SCRIPT_PATH="$VAULT_PATH/_system/bin/daily_ingest.py"
  LOG_PATH="$HOME/.cc-knowledge/ingest.log"
  PLIST_DEST="$HOME/Library/LaunchAgents/com.user.cc-knowledge.plist"

  sed -e "s|__PYTHON__|$PYTHON_PATH|g" \
      -e "s|__SCRIPT_PATH__|$SCRIPT_PATH|g" \
      -e "s|__API_KEY__|$API_KEY|g" \
      -e "s|__LOG_PATH__|$LOG_PATH|g" \
      "$SCRIPT_DIR/launchd/template.plist" > "$PLIST_DEST"

  launchctl load "$PLIST_DEST" && echo "launchd: loaded" || echo "Warning: launchctl load failed — load manually"
else
  echo "Note: launchd setup skipped (macOS only). Set up a cron job manually."
fi

# 9. Print summary
echo ""
echo "✓ cc-knowledge installed"
echo "  Vault:  $VAULT_PATH"
echo "  Config: $HOME/.cc-knowledge/config.json"
echo "  Run /o in Claude Code after a session to capture it."
echo "  Manual ingest: python3 $VAULT_PATH/_system/bin/daily_ingest.py"
```

**Helper scripts** (called by setup, live in `bin/`):

`bin/configure_mcp.py` — reads `~/.claude.json`, adds/updates `mcpServers` entry for filesystem server pointing at vault, writes back. Handles missing file (creates from scratch) and missing `mcpServers` key.

`bin/translate_scaffold.py` — calls `claude-sonnet-4-6` to translate `_system/index.md` headers/content into target language. Also renames natural-language files (e.g. `mistake-log.md`) using translation output. Writes a `_system/language.txt` file with the chosen language for runtime reference.

**Acceptance criteria:**
- Script is executable (`chmod +x setup`).
- Idempotent: re-running does not duplicate MCP entries or fail if vault already exists.
- Tilde expansion works in vault path.
- On Linux/WSL: launchd block is skipped, prints informational note.
- `~/.claude.json` `mcpServers` entry does not clobber other servers.

---

### Story 6: `SKILL.md` — `/o` slash command

**What:** Write the SKILL.md file that defines the `/o` command behavior for Claude Code.

**Structure:**

```markdown
# /o — Session Knowledge Capture

Summarize the current conversation and file it into your CC-Knowledge vault.

## Step 1: Read the index
Open `_system/index.md` via the filesystem MCP tool. Note existing concept pages and recap filenames.

## Step 2: Create session recap
Create `session-recaps/YYYY-MM-DD Title.md` with these sections:
...

## Step 3: Update concept library
...

## Step 4: Sync Q&As to qa-handbook
...

## Step 5: Update index
...

## Step 6: Write operation log
...

## Constraints
...
```

Full section behavior matches the spec exactly. Language note: "Write all content in the language configured for this vault (check `_system/language.txt` if present, otherwise English)."

**Acceptance criteria:**
- File exists at `SKILL.md` in repo root.
- All 6 steps documented.
- Q&A routing table maps to correct `qa-handbook/` files.
- Concept library update-first semantics documented.
- Language instruction present.

---

### Story 7: `README.md`

**What:** Write the installation and usage README.

**Sections:**
1. What is cc-knowledge (2 sentences)
2. Prerequisites (Python 3.10+, `pip install anthropic`, Claude Code)
3. macOS-only notice for launchd auto-ingest
4. Installation (`git clone`, `./setup`)
5. What you get (vault structure as tree)
6. Using `/o` after a session
7. Manual ingest (`--dry-run`, `--force <uuid>`, `--all`)
8. Config file (`~/.cc-knowledge/config.json`)
9. Uninstall instructions

**Acceptance criteria:**
- File exists at `README.md` in repo root.
- Install steps are copy-pasteable.
- macOS notice is prominent (second section).

---

### Story 8: Integration verification checklist

**What:** Manual verification steps to confirm the full system works end-to-end. This is not automated; it documents what the developer must check before marking the build complete.

**Checklist:**

- [ ] `./setup` runs without error on a clean macOS machine
- [ ] Vault directory created with correct structure
- [ ] `~/.cc-knowledge/config.json` written correctly
- [ ] `~/.claude.json` has new `mcpServers` entry (not duplicate)
- [ ] `com.user.cc-knowledge.plist` installed and visible via `launchctl list`
- [ ] `daily_ingest.py --dry-run` runs without import errors and finds JSONL files
- [ ] `daily_ingest.py --force <real-uuid>` generates a recap file in vault
- [ ] `/o` command in Claude Code reads index.md and creates a recap (manual test)
- [ ] Non-English setup: translated index.md headers appear in target language
- [ ] Re-running `./setup` is idempotent (no duplicate MCP entries, no overwrite errors)

---

## Execution Order

| # | Story | Depends On |
|---|-------|------------|
| 1 | Project scaffolding | — |
| 2 | scaffold/ template files | 1 |
| 3 | bin/daily_ingest.py | 1 |
| 4 | launchd/template.plist | 1 |
| 5 | setup bash script | 2, 3, 4 |
| 6 | SKILL.md | 2 |
| 7 | README.md | 5, 6 |
| 8 | Integration verification | 7 |

Stories 2, 3, and 4 can be worked in parallel after Story 1.
Stories 6 and 5 can be worked in parallel after Story 2 is done.

---

## File Map (final repo layout)

```
cc-knowledge/
├── SKILL.md
├── README.md
├── setup                          (executable bash)
├── bin/
│   ├── daily_ingest.py
│   ├── configure_mcp.py
│   └── translate_scaffold.py
├── scaffold/
│   ├── _system/
│   │   ├── index.md
│   │   ├── log.md
│   │   └── SCHEMA.md
│   ├── concept-library/.gitkeep
│   ├── session-recaps/.gitkeep
│   ├── methodology/.gitkeep
│   ├── engineering-insights/.gitkeep
│   ├── mistake-log/
│   │   └── mistake-log.md
│   ├── projects/.gitkeep
│   ├── open-source-library/
│   │   └── README.md
│   ├── weekly-digest/.gitkeep
│   └── qa-handbook/
│       ├── claude-code-ops.md
│       ├── git-github.md
│       └── general.md
├── launchd/
│   └── template.plist
└── .ship/
    └── ...
```

Total files to create: ~22 (excluding .gitkeep placeholders counted individually).
