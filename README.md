# agent-sync

Set up git-based multi-device sync for your AI agent (Hermes, OpenClaw, Claude Code, Codex, etc.).

Syncs config, skills, memories, cron jobs, and compact session notes across devices via git. No cloud drives. No daemons. No binary blobs.

## How to use

Tell your agent:

> Set up agent-sync from https://github.com/mastermindCodes/agent-sync

That's it. Agent clones the repo, reads the prompt, shows you what it'll do, asks confirmation, and builds everything.

Or paste the raw prompt directly:

```
https://raw.githubusercontent.com/mastermindCodes/agent-sync/main/PROMPT.md
```

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

If you want to force: `--update` (replace remote), `--mirror` (discard local), or `--merge` (auto-merge silently).

### How merge works

| Item | Merge strategy |
|------|---------------|
| config.yaml | Section by section, local keys win at key level |
| skills/ | Per-file, newer mtime wins. Both changed? Asks you |
| sessions/*.md | Append-only — concatenate, sort, dedupe. No conflicts |
| Everything else | If only on one side, it's included |

## Repo must be private

Your .env (API keys) is in sync by default. Use a **private** repo. If you don't want keys in git, tell the agent to exclude .env.

## Requirements

- git installed
- git-credential-manager (usually included with git) or SSH key set up for auth
- A private git repo (GitHub, GitLab, etc.)

## Why PROMPT.md instead of install.sh?

Because every agent and OS is different. A prompt tells the agent *what to do* — the agent reads your system, picks the right tools, and builds the sync setup tailored to you. One plaintext file works on Hermes, OpenClaw, Claude Code, Codex, and any agent that can read a file and run commands.
