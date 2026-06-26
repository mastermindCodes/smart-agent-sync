# agent-sync — Multi-Device Sync Setup

This prompt tells you (the AI agent) to set up git-based sync for this Hermes / OpenClaw / Claude Code / Codex / other AI agent installation.

**IMPORTANT: Before doing ANYTHING, show the user a summary of what will happen and ask for confirmation (Step 2a). Do not modify files or run commands until the user says yes.**

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

---

## Step 2: Show summary, ask confirmation, check git auth

**Step 2a: Show user what will happen**

Before doing anything, print a clear summary:

```
? agent-sync will:
  1. Detect your agent home directory
  2. Create a private git repo at <URL>
  3. Upload your config, skills, memories, cron, and session notes
  4. Set up 30-min auto-sync so other devices stay in sync
  5. Install session-summary hook (appends compact note on session end)

Your API keys in .env will be synced unless you say not to.
Your state.db, cache/, logs/ will NOT be synced (device-specific).

Ready to proceed? (y/n)
```

Wait for user confirmation. If they say no, stop and explain what to change.

If they want to exclude something, ask: "Anything you don't want synced? (e.g. `.env`, `certain_skill`)"

**Step 2b: Check git authentication**

Test if git can push:

```bash
git ls-remote <USER_REMOTE_URL> 2>&1
```

- If success (returns 0 or shows refs) — auth works, continue.
- If "Repository not found" — repo doesn't exist yet. Try creating it:
  - If `gh` CLI available: `gh repo create <name> --private`
  - If GitHub token available: create via API
  - Otherwise: tell user "Create a private repo at github.com/new named agent-sync, then come back."
- If auth error (403/401): "Git needs authentication. Your OS credential manager should pop up."
  - Retry after user authenticates.
  - If still fails: guide user to create PAT at github.com/settings/tokens and provide it.
- If no network: tell user to check connection.

**Step 2c: Ask for remote URL** (if not already provided by user)

Ask for URL only if user didn't include it in their first message.

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

## Step 4: Init git repo — Smart mode (default)

Smart is the default. No mode-picking needed. Agent detects the situation and acts. Only asks user if genuine conflict arises.

Also apply `$EXCLUDED_PATHS` (files/dirs user doesn't want synced — add to `.gitignore` before committing).

### Setup: get remote

```bash
cd "$AGENT_HOME"
git init
git checkout -b main
git remote add origin <USER_REMOTE_URL>
git fetch origin
```

### Case 1: Remote empty (first device) — upload local

```bash
git add -A
git commit -m "init: agent-sync"
git push -u origin main --force
```

If push fails auth: tell user OS credential manager will pop up, retry. If fails again, guide user to create PAT.

Done. Tell user: "Synced — your local config is now in the repo."

### Case 2: Remote has content (second device) — compare

```bash
git branch local-state main
git reset --hard origin/main
git branch remote-state origin/main
```

Walk each synced item and compare:

| Situation | Action |
|-----------|--------|
| Only on one side | Keep it (add to git) |
| Both sides, no diff | Skip (same) |
| Both sides, auto-mergeable (config keys, session notes) | Merge silently |
| Both sides, diff, not auto-mergeable | **Ask user** |

Show a diff summary with recommendation:

```
? Sync analysis for agent at $AGENT_HOME:
  config.yaml: 2 local keys not on remote, 1 remote key not local
    -> auto-merged (no conflicts)
  skills/code-review: on both, slightly different
    -> [m]erge both | [l]ocal wins | [r]emote wins
  cron/: only on remote -> keeping
  sessions/2026-06-26.md: local has 2 extra entries
    -> auto-appended (append-only)

Recommendation: merge all (keep best of both — no destructive conflicts)
Options: [m]erge all | [u]pload local (replace remote) | [r]estore from remote (mirror) | [c]ustom
```

After user choice:

```bash
git add -A
git commit -m "sync: merged local + remote"
git push -u origin main --force
git branch -D local-state remote-state 2>/dev/null || true
```

### Auto-merge rules

**config.yaml:** Merge section by section. Local keys win at key level. Commented lines preserved.

```python
import yaml
with open('local_config.yaml') as f: local = yaml.safe_load(f)
with open('remote_config.yaml') as f: remote = yaml.safe_load(f)
merged = {}
for section in set(list(local.keys()) + list(remote.keys())):
    lv = local.get(section, {})
    rv = remote.get(section, {})
    if isinstance(lv, dict) and isinstance(rv, dict):
        merged[section] = {**rv, **lv}
    else:
        merged[section] = lv if lv != rv else lv
with open('merged_config.yaml', 'w') as f:
    yaml.dump(merged, f)
```

**sessions/*.md:** Append-only. Concatenate, sort by timestamp, dedupe exact lines. No conflicts.

**skills/:** For same-named skill on both sides, compare mtime. One side changed? Keep that. Both changed? Flag user.

### Explicit flags (skip Smart, go direct)

**`--update`**: Force push local, replace remote:
```bash
cd "$AGENT_HOME" && git init && git checkout -b main && git add -A && git commit -m "agent-sync update" && git remote add origin <URL> 2>/dev/null; git fetch origin && git push -u origin main --force
```

**`--mirror`**: Discard local, clone remote:
```bash
mkdir -p /tmp/agent-sync-backup && cp "$AGENT_HOME"/sessions/*.md /tmp/agent-sync-backup/ 2>/dev/null; cd "$AGENT_HOME"/.. && mv "$(basename "$AGENT_HOME")" "$(basename "$AGENT_HOME")_old" && git clone <URL> "$(basename "$AGENT_HOME")" && cd "$AGENT_HOME" && rm -f state.db* sessions.db && rm -rf cache/ && cp /tmp/agent-sync-backup/*.md sessions/ 2>/dev/null && rm -rf /tmp/agent-sync-backup /tmp/"$(basename "$AGENT_HOME")_old" && mkdir -p cache/ logs/ audio_cache/
```

**`--merge`**: Auto-merge silently, local wins on conflict. Same logic as Case 2 but no prompts. Push when done.

---

## Step 5: Create sessions directory

```bash
mkdir -p "$AGENT_HOME/sessions"
```

This holds compact session summary notes — one markdown file per day, appended to on session end.

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
author: agent-sync
license: MIT
platforms: [linux, macos, windows]
---

# Session Summary

On `/new` (or any session end), append a compact summary note to
`$AGENT_HOME/sessions/YYYY-MM-DD.md` before session resets.

Format:

```
## YYYY-MM-DDTHH:MM — short title
- what: one line about what happened
- decisions: key decisions made
```

One entry per session. Append to the day's file. Skip trivial chat.

## Auto-sync

Every 30 minutes, commit and push:

```bash
cd "$AGENT_HOME"
git add -A
git diff --cached --quiet || (git commit -m "auto-sync: $(date +%Y-%m-%dT%H:%M)" && git push)
```
```

Tell user: "Run `/skills load hermes-sync` to activate. Or add to config.yaml: `skills: [hermes-sync]` for auto-load."

### For OpenClaw

Write session-summary hook to `$AGENT_HOME/hooks/post-session.sh`:

```bash
#!/bin/bash
AGENT_HOME="$HOME/.openclaw"
SESSION_NOTE="$AGENT_HOME/sessions/$(date +%Y-%m-%d).md"
mkdir -p "$(dirname "$SESSION_NOTE")"
echo "## $(date -u +%Y-%m-%dT%H:%M) — session ended" >> "$SESSION_NOTE"
echo "- what: $*" >> "$SESSION_NOTE"
echo "" >> "$SESSION_NOTE"
```

Make executable. Tell user to wire it into their agent's session-end callback.

### For Claude Code / Codex / Other

Find equivalent lifecycle hook (session_end, on_exit, post_prompt, etc.) and install matching script. If none exists, install a wrapper script and tell user to run it manually on session end.

---

## Step 7: Set up auto-sync schedule

Commit and push every 30 minutes. Detect the best scheduling mechanism.

### Priority (use first available):

**1. Hermes built-in cron (if this is Hermes):**

```python
cronjob(
    action="create",
    name="agent-sync",
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
cat > /tmp/agent-sync.service << 'SVC'
[Unit]
Description=agent-sync auto commit+push
[Service]
Type=oneshot
ExecStart=PATH_TO_AGENT_HOME/sessions/auto-sync.sh
User=CHANGE_THIS_TO_YOUR_USERNAME
SVC

cat > /tmp/agent-sync.timer << 'TMR'
[Unit]
Description=agent-sync every 30min
[Timer]
OnCalendar=*:0/30
Persistent=true
[Install]
WantedBy=timers.target
TMR

sudo mv /tmp/agent-sync.service /etc/systemd/system/
sudo mv /tmp/agent-sync.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now agent-sync.timer
```

Replace `CHANGE_THIS_TO_YOUR_USERNAME` and `PATH_TO_AGENT_HOME` with actual values.

**3. launchd (macOS):**

```bash
cat > ~/Library/LaunchAgents/com.agent-sync.plist << 'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.agent-sync</string>
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
launchctl load ~/Library/LaunchAgents/com.agent-sync.plist
```

**4. Windows Task Scheduler:**

Detect git-bash path first (common: `C:\Program Files\Git\bin\bash.exe`):

```powershell
$gitBash = "C:\Program Files\Git\bin\bash.exe"
$script = "-c 'cd $env:USERPROFILE\.hermes && git add -A && git diff --cached --quiet || (git commit -m auto-sync && git push)'"
$action = New-ScheduledTaskAction -Execute $gitBash -Argument $script
$trigger = New-ScheduledTaskTrigger -Daily -RepetitionInterval (New-TimeSpan -Minutes 30) -RepetitionDuration (New-TimeSpan -Days 365)
Register-ScheduledTask -TaskName "agent-sync" -Action $action -Trigger $trigger -RunLevel Highest
```

If git-bash not found, try `wsl.exe git` or fall back to PowerShell-only script using git from PATH.

**5. crontab (last resort):**

```bash
(crontab -l 2>/dev/null; echo "*/30 * * * * cd $AGENT_HOME && git add -A && git diff --cached --quiet || (git commit -m 'auto-sync: $(date -u +\%Y-\%m-\%dT\%H:\%M)' && git push -q 2>/dev/null || true)") | crontab -
```

---

## Step 8: Report to user

Print summary:

```
? agent-sync installed

Agent home: $AGENT_HOME
Mode: Smart (<uploaded local / merged / mirrored>)
Git remote: <USER_REMOTE_URL>
Auto-sync: every 30 minutes (via <method>)
Session notes: $AGENT_HOME/sessions/
Gitignore: $AGENT_HOME/.gitignore
Excluded: <anything user opted out of>

Session summary hook: <how it was installed>

On your other devices, run this same prompt.
Smart mode detects it's a second device and offers merge.

! .env is INCLUDED in git (API keys). To exclude:
   echo '.env' >> $AGENT_HOME/.gitignore
   git rm --cached .env
   git commit -m "stop syncing .env"

Next steps:
  - Test: cd $AGENT_HOME && git add -A && git commit -m "test" && git push
  - On other device: run this prompt — Smart handles everything
```

---

## Notes for the implementing agent

- **Repo is private.** Tell user. Public repos leak API keys.
- **User can say "exclude X"** — add to `.gitignore`, `git rm --cached` before first commit.
- **Do not delete user data.** Mirror mode removes only state.db, cache/, logs/ (regenerated). Merge keeps everything.
- **Smart mode asks only on real conflicts.** Auto-merge everything else silently.
- **config.yaml merge:** section by section, local keys win at key level.
- **sessions/*.md merge:** concatenate, sort by timestamp, dedupe lines. No conflicts possible.
- **skills/ merge:** per-file mtime wins. Both changed? Flag user.
- **If push fails auth:** OS credential manager pops up. Authenticate once. Retry.
- **If no credential helper:** guide user: `git config --global credential.helper <helper>` or SSH key.
- **Session-summary script must NOT block or prompt.** Silent append on session end.
- **If no lifecycle hook found** (unfamiliar agent): install wrapper script, tell user to run before closing session.
