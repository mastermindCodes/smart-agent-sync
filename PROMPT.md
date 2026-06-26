# agent-sync — Multi-Device Sync Setup

This prompt tells you (the AI agent) to set up git-based sync for this Hermes / OpenClaw / Claude Code / Codex / other AI agent installation.

Follow these instructions exactly. Do not skip steps. Do not ask questions — figure it out from the system.

---

## Overview

Create a git repo inside the agent's home directory. Sync config, skills, memories, cron jobs, and compact session notes across devices. No cloud drives. Auto-commit+push every 30 minutes. Append session summaries on session end.

---

## Step 1: Detect the agent home directory

Try these in order until you find one that exists:

- **Hermes:** `python -c "from hermes_constants import get_hermes_home; print(get_hermes_home())"` — if that module isn't found, check environment variable `HERMES_HOME`, then fall back to `~/.hermes` (POSIX) or `%LOCALAPPDATA%/hermes` (Windows)
- **OpenClaw:** Check `OPENCLAW_HOME` env var, then `~/.openclaw/`
- **Claude Code:** Check `~/.claude/` or `$XDG_CONFIG_HOME/claude/`
- **Codex:** Check `CODEX_HOME` env var, then `~/.codex/`
- **Generic:** Check `~/.config/` for any agent config dirs, or ask the user

Set `AGENT_HOME` to the resolved path. If you can't find it, ask the user where their agent stores data.

**Clone detection:** If the user passes `--clone` as an argument to their prompt (e.g. "hermes-sync --clone"), set `IS_CLONE=true`. Otherwise `IS_CLONE=false`.

---

## Step 2: Create .gitignore

Write this file at `$AGENT_HOME/.gitignore`:

```
# Binaries, DBs, caches — device-specific, don't sync
state.db
state.db*
sessions.db
sessions.db*
kanban.db
kanban.db*
verification_evidence.db
cache/
audio_cache/
image_cache/
logs/
sandboxes/
bin/

# Lock files and temp
*.lock
*.pid
*.bak
*.bak.*
*.swp
gateway_state.json
channel_directory.json
gateway.lock
interrupt_debug.log
processes.json

# Large model caches (regenerated per device)
models_dev_cache.json
ollama_cloud_models_cache.json
provider_models_cache.json

# Credentials — don't sync .env unless user opts in
# Uncomment the line below if user does NOT want API keys in git
# .env
```

---

## Step 3: Init git repo or clone

Check if `$AGENT_HOME` is already a git repo (`git -C "$AGENT_HOME" rev-parse --git-dir`).

**If NOT a git repo AND IS_CLONE is false:**

```bash
cd "$AGENT_HOME"
git init
git checkout -b main
git add -A
git commit -m "init: hermes-sync base state"
```

Ask the user for their git remote URL (e.g. `git@github.com:user/hermes-sync.git` or `https://github.com/user/hermes-sync.git`). Then:

```bash
git remote add origin <URL>
git push -u origin main
```

If push fails with an auth error, tell the user: *"Git push requires authentication. Your OS credential manager should pop up — authenticate once, then I'll retry."* Then retry the push. If it fails again, guide the user to set up a Personal Access Token.

**If ALREADY a git repo or IS_CLONE is true:**

If `$AGENT_HOME` is already a git repo with a remote set, pull the latest:

```bash
cd "$AGENT_HOME"
git fetch origin
git reset --hard origin/main
# Remove device-specific files that shouldn't be overwritten from remote
rm -f state.db state.db-shm state.db-wal sessions.db 2>/dev/null
rm -rf cache/ logs/ sandboxes/ audio_cache/ image_cache/ bin/ 2>/dev/null
```

If it's not a git repo yet (clone scenario), clone fresh:

```bash
# Move existing data aside temporarily (user's current sessions, etc.)
backup_dir=$(mktemp -d)
cp -r "$AGENT_HOME/config.yaml" "$AGENT_HOME/.env" "$AGENT_HOME/skills" "$AGENT_HOME/memories" "$AGENT_HOME/cron" "$backup_dir/" 2>/dev/null || true

# Clone
cd "$(dirname "$AGENT_HOME")" && git clone <URL> "$(basename "$AGENT_HOME")" 2>/dev/null || \
  (cd "$AGENT_HOME" && git init && git remote add origin <URL> && git fetch origin && git reset --hard origin/main)

# Restore local data that isn't in git yet, then add+commit
cp -r "$backup_dir/"* "$AGENT_HOME/" 2>/dev/null || true
rm -rf "$backup_dir"
rm -f "$AGENT_HOME/state.db" "$AGENT_HOME/state.db-shm" "$AGENT_HOME/state.db-wal" "$AGENT_HOME/sessions.db" 2>/dev/null
rm -rf "$AGENT_HOME/cache/" "$AGENT_HOME/logs/" "$AGENT_HOME/sandboxes/" "$AGENT_HOME/audio_cache/" "$AGENT_HOME/image_cache/" "$AGENT_HOME/bin/" 2>/dev/null
```

If no remote is provided (user hasn't created a git repo yet in clone scenario), fall back to the init path above and tell the user to create a repo first.

---

## Step 4: Create sessions directory

```bash
mkdir -p "$AGENT_HOME/sessions"
```

This directory holds compact session summary notes — one markdown file per day, appended to on session end.

---

## Step 5: Install session-summary hook

Figure out which agent this is. Create the appropriate hook.

### For Hermes

Create a skill at `$AGENT_HOME/skills/hermes-sync/SKILL.md`:

```markdown
---
name: hermes-sync
description: "Append session summary on /new and auto-sync to git."
version: 1.0.0
author: hermes-sync
license: MIT
platforms: [linux, macos, windows]
---

# Hermes Sync Session Summary

On `/new` (or any session end), you MUST append a compact summary note to
`$AGENT_HOME/sessions/YYYY-MM-DD.md` before the session resets.

The format is:

```
## YYYY-MM-DDTHH:MM — short title
- what: one line about what happened
- decisions: key decisions made
```

Only note things worth remembering across sessions. Skip trivial chat.
One entry per session. Append to the day's file, don't overwrite.

## Auto-sync

Every 30 minutes, commit and push any changes:

```bash
cd "$AGENT_HOME"
git add -A
git diff --cached --quiet || (git commit -m "auto-sync: $(date +%Y-%m-%dT%H:%M)" && git push)
```

Then tell the user: *"Run `/skills load hermes-sync` to activate. Or add to config.yaml: `skills: [hermes-sync]` for auto-load on every session."*

### For OpenClaw

Write the session-summary hook to `$AGENT_HOME/hooks/post-session.sh` (or the equivalent lifecycle hook path):

```bash
#!/bin/bash
AGENT_HOME="$HOME/.openclaw"
SESSION_NOTE="$AGENT_HOME/sessions/$(date +%Y-%m-%d).md"
mkdir -p "$(dirname "$SESSION_NOTE")"
echo "## $(date -u +%Y-%m-%dT%H:%M) — session ended" >> "$SESSION_NOTE"
echo "- what: $*" >> "$SESSION_NOTE"
echo "" >> "$SESSION_NOTE"
```

Make it executable. Tell the user to wire it into their agent's session-end callback.

### For Claude Code / Codex / Other

Find their equivalent lifecycle hook (session_end, on_exit, post_prompt, etc.) and install a matching script. If none exists, install a wrapper script and tell the user to run it manually on session end, or use the cron-based auto-commit as a fallback (Step 6).

---

## Step 6: Set up auto-sync schedule

Commit and push every 30 minutes. Detect the best scheduling mechanism.

### Priority (use first available):

**1. Hermes built-in cron (if this is Hermes):**

```python
# Use the cronjob tool to create a no_agent script job
cronjob(
    action="create",
    name="hermes-sync",
    schedule="*/30 * * * *",
    script="AGENT_HOME/sessions/auto-sync.sh".replace("AGENT_HOME", AGENT_HOME),
    no_agent=True
)
```

Create the script first:

```bash
cat > "$AGENT_HOME/sessions/auto-sync.sh" << 'SCRIPT'
#!/bin/bash
cd "$AGENT_HOME"
git add -A
git diff --cached --quiet && exit 0
git commit -m "auto-sync: $(date -u +%Y-%m-%dT%H:%M)"
git push -q 2>/dev/null || true
SCRIPT
chmod +x "$AGENT_HOME/sessions/auto-sync.sh"
```

**2. systemd timer (Linux with systemd):**

```bash
cat > /tmp/hermes-sync.service << 'SVC'
[Unit]
Description=hermes-sync auto commit+push
[Service]
Type=oneshot
ExecStart=PATH_TO_AGENT_HOME/sessions/auto-sync.sh
User=CHANGE_THIS_TO_YOUR_USERNAME
SVC

cat > /tmp/hermes-sync.timer << 'TMR'
[Unit]
Description=hermes-sync every 30min
[Timer]
OnCalendar=*:0/30
Persistent=true
[Install]
WantedBy=timers.target
TMR

sudo mv /tmp/hermes-sync.service /etc/systemd/system/
sudo mv /tmp/hermes-sync.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now hermes-sync.timer
```

Replace `CHANGE_THIS_TO_YOUR_USERNAME` and `PATH_TO_AGENT_HOME` with actual values before running.

**3. launchd (macOS):**

```bash
cat > ~/Library/LaunchAgents/com.hermes-sync.plist << 'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.hermes-sync</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>-c</string>
        <string>cd $HOME/<AGENT_DIR> && git add -A && git diff --cached --quiet || (git commit -m "auto-sync: $(date -u +%Y-%m-%dT%H:%M)" && git push -q 2>/dev/null || true)</string>
    </array>
    <key>StartInterval</key>
    <integer>1800</integer>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
PLIST
launchctl load ~/Library/LaunchAgents/com.hermes-sync.plist
```

**4. Windows Task Scheduler:**

Detect the git-bash `bash.exe` path first (common: `C:\Program Files\Git\bin\bash.exe`), then:

```powershell
$gitBash = "C:\Program Files\Git\bin\bash.exe"
$script = "-c 'cd $env:USERPROFILE\.hermes && git add -A && git diff --cached --quiet || (git commit -m auto-sync && git push)'"
$action = New-ScheduledTaskAction -Execute $gitBash -Argument $script
$trigger = New-ScheduledTaskTrigger -Daily -RepetitionInterval (New-TimeSpan -Minutes 30) -RepetitionDuration (New-TimeSpan -Days 365)
Register-ScheduledTask -TaskName "hermes-sync" -Action $action -Trigger $trigger -RunLevel Highest
```

If git-bash not found, try `wsl.exe git` or `C:\Windows\System32\bash.exe` (WSL). If none of those exist, fall back to a PowerShell-only script using git from PATH:

**5. crontab (last resort — works everywhere):**

```bash
(crontab -l 2>/dev/null; echo "*/30 * * * * cd $AGENT_HOME && git add -A && git diff --cached --quiet || (git commit -m 'auto-sync: $(date -u +\%Y-\%m-\%dT\%H:\%M)' && git push -q 2>/dev/null || true)") | crontab -
```

---

## Step 7: First local commit after clone

If this is a second device (IS_CLONE was true):

```bash
cd "$AGENT_HOME"
git add -A
git diff --cached --quiet || git commit -m "sync: updates from $(hostname)"
git push
```

---

## Step 8: Report to user

Print a summary of what was set up:

```
✅ hermes-sync installed

Agent home: $AGENT_HOME
Git remote: <remote URL>
Auto-sync: every 30 minutes (via <cron/systemd/launchd/task-scheduler>)
Session notes: $AGENT_HOME/sessions/
Gitignore: $AGENT_HOME/.gitignore

Session summary hook: <how it was installed>

On your other devices, run the same prompt. Your git remote already exists,
so the agent will clone instead of init.

⚠️  .env is INCLUDED in git (API keys). If you don't want secrets in git:
   echo '.env' >> $AGENT_HOME/.gitignore && git rm --cached .env && git commit -m "stop syncing .env"

Next steps:
  - Make a test commit: cd $AGENT_HOME && git add -A && git commit -m "test sync" && git push
  - On other device: run the same prompt (agent detects it should clone)
```

---

## Notes for the implementing agent

- **Do not delete or overwrite user data.** The clone step for second devices removes only `state.db`, `cache/`, `logs/` etc. — these are regenerated.
- **If `git push` asks for credentials**, let the OS credential manager pop up. On macOS it's `osxkeychain`, on Windows it's `git-credential-manager`, on Linux it's `libsecret` or `manager-core`. The user authenticates once.
- **If no credential helper is configured**, tell the user to set one up: `git config --global credential.helper <helper>` or use SSH (`ssh-keygen -t ed25519` + add to GitHub).
- **The session-summary script must NOT block or prompt.** It runs silently on session end.
- **If you can't find an agent lifecycle hook** (e.g. for an unfamiliar agent), fall back to a wrapper script and tell the user to run it manually before closing their session.
