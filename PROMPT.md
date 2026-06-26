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

**Clone detection:** Check if `$AGENT_HOME` already has significant config (config.yaml, skills/ with files). If yes, this is likely a second device. Also check if user says "second device", "clone", or "mirror".

Modes (ask the user if not obvious from context):
- **merge** (default): Compare local and remote config, merge intelligently, keep the best of both
- **mirror**: Discard local state, pull exactly what's on remote (pure backup restore)
- **override**: Discard remote, push local as new truth (first device setup)

---

## Step 2: Ask user for git remote + mode

Ask the user:

> *"I'll set up git sync for your agent. I need:*
> 1. *Git remote URL (e.g. `https://github.com/user/agent-sync.git`)*
> 2. *Repo will be **private** — your API keys live there.*
> 3. *Mode: **merge** (combine best from both), **mirror** (copy from remote), or **override** (push local)?*
> 4. *Anything you DON'T want synced? (e.g. `.env`, `certain_skill`)*"

If user says nothing about mode, default to **merge** for second device, **override** for first.

---

## Step 3: Create .gitignore

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

## Step 4: Init git repo — three modes

You already have the remote URL and mode from Step 2. Also have `$EXCLUDED_PATHS` (list of files/dirs user doesn't want synced — remove them from `.gitignore` before committing).

### Mode A: override (first device — push local as truth)

```bash
cd "$AGENT_HOME"
git init
git checkout -b main
git add -A
git commit -m "init: agent-sync base state"
git remote add origin <USER_REMOTE_URL>
git push -u origin main
```

If push fails with auth error: *"Your OS credential manager should pop up — authenticate once, then I'll retry."* Retry. If fails again, guide user to create a Personal Access Token.

### Mode B: mirror (second device — discard local, copy remote)

```bash
cd "$AGENT_HOME"
# Backup anything user might want locally
mkdir -p /tmp/agent-sync-backup
cp sessions/*.md /tmp/agent-sync-backup/ 2>/dev/null || true

# Wipe and clone fresh
cd ..
mv "$(basename "$AGENT_HOME")" "$(basename "$AGENT_HOME")_old" 2>/dev/null || true
git clone <USER_REMOTE_URL> "$(basename "$AGENT_HOME")"
cd "$AGENT_HOME"

# Restore device-specific files (NOT config, skills, memories — those come from remote)
rm -f state.db state.db-shm state.db-wal sessions.db cache/ -rf 2>/dev/null
cp /tmp/agent-sync-backup/*.md sessions/ 2>/dev/null || true
rm -rf /tmp/agent-sync-backup /tmp/"$(basename "$AGENT_HOME")_old" 2>/dev/null || true

# Regenerate device-specific data
mkdir -p cache/ logs/ audio_cache/ 2>/dev/null
```

### Mode C: merge (second device — keep best of both — DEFAULT)

This is the smart one. For each synced directory, compare local vs remote. For text files that are both new (skills, config), diff and merge. For same keys, newer wins. For conflicts, ask user.

**Step C1: Get remote into a temp branch**

```bash
cd "$AGENT_HOME"
git init
git checkout -b main
git remote add origin <USER_REMOTE_URL>
git fetch origin

# If remote has content
if git rev-parse origin/main >/dev/null 2>&1; then
    git branch local-state main
    git reset --hard origin/main
    git branch remote-state origin/main
```

**Step C2: For each synced item, merge intelligently**

For each file/dir in the sync list (config.yaml, .env, skills/, memories/, cron/, sessions/, SOUL.md, plugins/):

| Situation | Action |
|-----------|--------|
| Only on remote, not local | Keep remote (it's new) |
| Only on local, not remote | Add to git, commit as new |
| Both exist, text file | `git diff local-state..remote-state -- <file>` — if only one side changed, keep that. If both changed, three-way merge |
| Both exist, skill dir | Walk each skill file. Same logic: newer per-file |
| Both exist, config.yaml | Section-level merge: newer per-key. Commented lines keep their comments |
| Both exist, sessions/*.md | Both are append-only — concatenate, sort by timestamp, dedupe. Append-only means no conflict |
| Binary / DB file | Skip (they're in .gitignore anyway) |

Use Python or the agent's tools to do per-key config merging:

```python
# Example: merge config.yaml section by section
import yaml, json
with open('local_config.yaml') as f: local = yaml.safe_load(f)
with open('remote_config.yaml') as f: remote = yaml.safe_load(f)

merged = {}
for section in set(list(local.keys()) + list(remote.keys())):
    lv = local.get(section, {})
    rv = remote.get(section, {})
    if isinstance(lv, dict) and isinstance(rv, dict):
        merged[section] = {**rv, **lv}  # local keys override remote
    elif lv != rv:
        merged[section] = lv  # prefer local for scalar conflicts
    else:
        merged[section] = lv

with open('merged_config.yaml', 'w') as f:
    yaml.dump(merged, f)
```

For skill files: walk `skills/` directories. If a skill exists in both, compare each file — newer modification time wins. If same mtime, content diff — if different, ask user or auto-pick local.

**Step C3: Commit merged state**

```bash
git add -A
git commit -m "merge: combined local and remote state"
git push -u origin main
```

Remove temp branches:

```bash
git branch -D local-state remote-state 2>/dev/null || true
```

### Handle excluded paths

Before every commit, remove user's excluded paths from tracking:

```bash
echo "$EXCLUDED_PATHS" | while read path; do
    echo "$path" >> .gitignore
    git rm --cached "$path" 2>/dev/null || true
done
git add -A
```

### Auth fallback (all modes)

If push fails: *"Git push needs authentication. OS credential manager will pop up."* Retry. If fails again, guide user to create PAT at github.com/settings/tokens.

---

## Step 5: Create sessions directory

```bash
mkdir -p "$AGENT_HOME/sessions"
```

This directory holds compact session summary notes — one markdown file per day, appended to on session end.

---

## Step 6: Install session-summary hook

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

Find their equivalent lifecycle hook (session_end, on_exit, post_prompt, etc.) and install a matching script. If none exists, install a wrapper script and tell the user to run it manually on session end, or use the cron-based auto-commit as a fallback (Step 7).

---

## Step 7: Set up auto-sync schedule

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

## Step 8: First local commit after clone

If mode was **mirror** or **merge**: this step is already done as part of Step 4.

If mode was **mirror**: no extra commit needed, remote is the source of truth.

---

## Step 9: Report to user

Print a summary of what was set up:

```
✅ agent-sync installed

Agent home: $AGENT_HOME
Mode: <override/mirror/merge>
Git remote: <USER_REMOTE_URL>
Auto-sync: every 30 minutes (via <built-in cron/systemd/launchd/task-scheduler/crontab>)
Session notes: $AGENT_HOME/sessions/
Gitignore: $AGENT_HOME/.gitignore
Excluded: <anything user opted out of>

Session summary hook: <how it was installed>

On your other devices, run the same prompt.
The agent will detect it's a second device and offer merge (default) or mirror mode.

⚠️  .env is INCLUDED in git (API keys). If you don't want secrets in git:
   echo '.env' >> $AGENT_HOME/.gitignore && git rm --cached .env && git commit -m "stop syncing .env"

Next steps:
  - Make a test commit: cd $AGENT_HOME && git add -A && git commit -m "test sync" && git push
  - On other device: run this same prompt — agent handles merge/mirror automatically
```

---

## Notes for the implementing agent

- **Repo is private.** Tell user to create a private repo. Mention public repos leak API keys.
- **User can say "exclude X"** — add to `.gitignore` and `git rm --cached` before first commit.
- **Do not delete or overwrite user data.** Mirror mode removes only `state.db`, `cache/`, `logs/` etc. — these are regenerated. Merge mode keeps everything.
- **If `git push` asks for credentials**, let the OS credential manager pop up. On macOS it's `osxkeychain`, on Windows it's `git-credential-manager`, on Linux it's `libsecret` or `manager-core`. The user authenticates once.
- **If no credential helper is configured**, tell the user to set one up: `git config --global credential.helper <helper>` or use SSH (`ssh-keygen -t ed25519` + add to GitHub).
- **The session-summary script must NOT block or prompt.** It runs silently on session end.
- **If you can't find an agent lifecycle hook** (e.g. for an unfamiliar agent), fall back to a wrapper script and tell the user to run it manually before closing their session.
- **Merge mode: config.yaml is merged per-section, skills are merged per-file, sessions/*.md are concatenated and deduped.** This is the key feature — make it work well.
