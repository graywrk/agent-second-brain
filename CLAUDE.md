# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Voice-first personal AI assistant for Telegram. Users send voice/text/photos to a Telegram bot, which saves entries to an Obsidian vault, processes them with Claude Code CLI (`--print` mode), creates tasks in Todoist via MCP, and sends HTML reports back to Telegram.

**Language:** Russian (UI, prompts, reports). Code and comments in English.

## Build & Run

```bash
# Install dependencies
uv sync

# Install dev dependencies
uv sync --group dev

# Run the bot
uv run python -m d_brain

# Run via systemd (production)
sudo systemctl start d-brain-bot
sudo journalctl -u d-brain-bot -f
```

## Linting & Type Checking

```bash
uv run ruff check src/              # Lint
uv run ruff check --fix src/        # Auto-fix
uv run mypy src/                    # Type check (strict mode)
```

Ruff config: line-length 88, target Python 3.12, rules: E, F, I, B, UP.

## Tests

```bash
uv run pytest                       # All tests
uv run pytest tests/test_foo.py     # Single file
uv run pytest -k "test_name"        # Single test
```

pytest-asyncio with `asyncio_mode = "auto"` (no need for `@pytest.mark.asyncio`).

## Architecture

### Core Flow

```
Telegram → aiogram Bot → Handlers → Services → Vault (Obsidian files)
                                       ↓
                              Claude CLI (--print) → MCP (Todoist)
                                       ↓
                              HTML report → Telegram
```

### Package Structure (`src/d_brain/`)

- `config.py` — Pydantic Settings loaded from `.env`
- `__main__.py` — Entry point, calls `bot.main.run_bot()`
- `bot/main.py` — Bot and Dispatcher creation, auth middleware. **Router order matters** (text router must be last as catch-all)
- `bot/handlers/` — One file per message type: `voice`, `text`, `photo`, `forward`, `commands`, `buttons`, `process`, `do`, `weekly`
- `bot/states.py` — FSM states (for `/do` command's two-step flow)
- `bot/formatters.py` — Format processing results for Telegram
- `bot/keyboards.py` — Reply keyboard markup
- `services/processor.py` — **Key service.** Shells out to `claude --print --dangerously-skip-permissions --mcp-config` for daily processing, arbitrary requests, and weekly digests
- `services/storage.py` — Appends entries to `vault/daily/YYYY-MM-DD.md`
- `services/transcription.py` — Deepgram Nova-3 async transcription (Russian)
- `services/session.py` — JSONL append-only session log at `vault/.sessions/{user_id}.jsonl`
- `services/git.py` — Git add/commit/push automation

### Vault Structure (`vault/`)

| Path | Purpose |
|------|---------|
| `daily/YYYY-MM-DD.md` | Raw daily entries |
| `goals/` | Goal cascade: 3-year vision → yearly → monthly → weekly |
| `thoughts/` | Processed categorized notes |
| `MOC/` | Maps of Content indexes |
| `attachments/` | Photos organized by date |
| `summaries/` | Weekly digest archives |
| `.sessions/` | JSONL session logs |
| `.claude/` | Claude skills, agents, rules (see below) |

### Claude Skills (in `vault/.claude/skills/`)

- `dbrain-processor` — Main daily processing skill with reference files (about, classification, todoist rules, goals)
- `graph-builder` — Vault link analysis (`scripts/analyze.py`, `scripts/add_links.py`)
- `todoist-ai` — Todoist MCP integration reference

### Automation (`scripts/`, `deploy/`)

- `scripts/process.sh` — Cron-triggered daily processing: runs Claude CLI from vault dir, rebuilds graph, git commits, sends report to Telegram
- `scripts/weekly.py` — Weekly digest generation
- `deploy/` — systemd units: `d-brain-bot.service` (main bot), `d-brain-process.{service,timer}` (daily at 21:00), `d-brain-weekly.{service,timer}` (Sunday 20:00)

## Key Design Decisions

- **Claude CLI subprocess:** Processing uses `subprocess.run(["claude", "--print", ...])` not the API. This runs in a thread via `asyncio.to_thread()` to avoid blocking the event loop. Default timeout is 1200s (20 min).
- **MCP for Todoist:** Claude accesses Todoist via MCP stdio server (`@doist/todoist-ai`). The `TODOIST_API_KEY` env var is passed to the subprocess.
- **HTML output:** All Claude outputs must be raw Telegram HTML (`<b>`, `<i>`, `<code>` tags only). No Markdown.
- **Vault as context:** When `scripts/process.sh` runs, it `cd`s into `vault/` so Claude reads `vault/.claude/CLAUDE.md` for full context. The bot's `ClaudeProcessor` runs from project root with `--mcp-config`.

## Environment Variables

See `.env.example`. Required: `TELEGRAM_BOT_TOKEN`, `DEEPGRAM_API_KEY`. Optional: `TODOIST_API_KEY`, `VAULT_PATH` (default `./vault`), `ALLOWED_USER_IDS` (JSON array).
