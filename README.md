# agent-sync

Set up git-based multi-device sync for your AI agent (Hermes, OpenClaw, Claude Code, Codex, etc.).

Syncs config, skills, memories, cron jobs, and compact session notes across devices via git. No cloud drives. No daemons. No binary blobs.

## How it works

One prompt. You paste it into your agent. The agent reads your system and sets everything up.

```bash
# Option 1: copy URL, open in browser, copy raw content
curl -fsSL https://raw.githubusercontent.com/mastermindCodes/agent-sync/main/PROMPT.md

# Option 2: pipe directly into clipboard
curl -fsSL https://raw.githubusercontent.com/mastermindCodes/agent-sync/main/PROMPT.md | pbcopy  # macOS
curl -fsSL https://raw.githubusercontent.com/mastermindCodes/agent-sync/main/PROMPT.md | clip    # Windows
curl -fsSL https://raw.githubusercontent.com/mastermindCodes/agent-sync/main/PROMPT.md | xclip   # Linux

# Then paste into your agent
```

Or just open the [raw PROMPT.md](https://raw.githubusercontent.com/mastermindCodes/agent-sync/main/PROMPT.md) and copy-paste.

## What syncs

| What | Why |
|------|-----|
| config.yaml | Settings |
| .env | API keys (optional — skip if paranoid) |
| skills/ | Agent-created procedural memory |
| memories/ | What your agent remembers about you |
| cron/ | Scheduled jobs |
| sessions/*.md | Compact session summaries (one file per day) |
| SOUL.md | Personality config |

Ignores: state.db, sessions.db, cache/, logs/, *.bak, *.lock (all regenerated or machine-specific).

## Device setup

**Device 1 (override mode):** Run the prompt. Agent creates git repo, asks for remote URL, pushes your local config as truth.

**Device 2+ (merge mode — default):** Run the same prompt on your other machine. Agent detects existing local config, fetches remote, and **intelligently merges** both sides:
- `config.yaml`: merged per-section, local keys win on conflict
- `skills/`: merged per-file, newer mtime wins
- `sessions/*.md`: concatenated and deduped by timestamp
- Everything else: if only on one side, it's included

You can also pick **mirror mode** (discard local, copy remote) if you want a clean clone.

## Repo must be private

Your `.env` (API keys) is in sync by default. Use a **private** GitHub/GitLab repo. If you don't want API keys in git, tell the agent to exclude `.env`.

## How session summaries work

On `/new` (or session end), your agent writes a compact note to `sessions/YYYY-MM-DD.md`:

```
## 2026-06-26T17:30 — fixed DB migration bug
- what: alembic version table not seeded on fresh DB
- fix: stamp head in init script
- files: scripts/init_db.sh, alembic/env.py
- decisions: skip CI squash this time
```

One file per day. Append-only. Git-friendly. No merge conflicts.

## Auto-sync

Agent sets up a 30-minute auto-commit+push schedule using whatever cron system your platform has (systemd timer, launchd, Task Scheduler, or the agent's built-in cron if available).

## Requirements

- git installed
- git-credential-manager (usually included with git) or SSH key set up for auth
- A private git repo (GitHub, GitLab, etc.)

## Why PROMPT.md instead of install.sh?

Because every agent and OS is different. A prompt tells the agent *what to do* — the agent reads your system, picks the right tools, and builds the sync setup tailored to you. One plaintext file works on Hermes, OpenClaw, Claude Code, Codex, and any agent that can read a file and run commands.
