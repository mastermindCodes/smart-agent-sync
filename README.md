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

## Smart mode (default)

No mode-picking. Agent figures it out:

| Situation | What happens |
|-----------|-------------|
| First device, repo empty | Uploads local config to git |
| Second device, repo has data | Pulls remote, compares local vs remote, auto-merges what it can, **asks you only on real conflicts** |

If you want to force a specific behavior: `--update` (replace remote), `--mirror` (discard local), or `--merge` (auto-merge silently).

### How merge works

| Item | Merge strategy |
|------|---------------|
| config.yaml | Section by section, local keys win at key level |
| skills/ | Per-file, newer mtime wins. Both changed? Asks you |
| sessions/*.md | Append-only — concatenate, sort, dedupe. No conflicts |
| Everything else | If only on one side, it's included |

## Device setup

1. Paste PROMPT.md into your agent on device 1
2. Give it a git remote URL
3. Agent does the rest — creates repo, pushes, sets up session summaries + auto-sync
4. On device 2: paste same prompt. Smart mode detects existing remote, merges.

## Repo must be private

Your .env (API keys) is in sync by default. Use a **private** repo. If you don't want keys in git, tell the agent to exclude .env.

## Requirements

- git installed
- git-credential-manager (usually included with git) or SSH key set up for auth
- A private git repo (GitHub, GitLab, etc.)

## Why PROMPT.md instead of install.sh?

Because every agent and OS is different. A prompt tells the agent *what to do* — the agent reads your system, picks the right tools, and builds the sync setup tailored to you. One plaintext file works on Hermes, OpenClaw, Claude Code, Codex, and any agent that can read a file and run commands.
