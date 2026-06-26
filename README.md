# smart-agent-sync

One prompt. Paste into your AI agent. Git-based multi-device sync for config, skills, memories, and session notes.

```
Set up smart-agent-sync from https://github.com/mastermindCodes/smart-agent-sync
```

## How it works

Agent reads [PROMPT.md](./PROMPT.md) — explains what'll happen, checks git auth, creates repo or pulls existing, merges smartly, sets up 30-min auto-sync. Works on Hermes, OpenClaw, Claude Code, Codex, any agent.

**Smart mode** — first device uploads, second device compares + merges + asks only on conflicts.

[GPLv3](./LICENSE)
