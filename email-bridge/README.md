# Email Bridge for Claude Code tmux Sessions

Sends email notifications from Claude Code hooks and accepts commands via email replies, routed to tmux sessions.

## Architecture

```
Claude Code hooks → events.jsonl → bridge.py daemon → Gmail (SMTP)
Gmail (IMAP) → bridge.py → tmux send-keys → Claude session
```

## Files

All live in `~/.claude/email-bridge/`:

| File | Purpose |
|------|---------|
| `config.json` | Gmail creds, poll intervals (fill in before first use) |
| `hook_notify.py` | Hook script — appends events to queue |
| `bridge.py` | Main daemon — email send/receive + tmux routing |
| `launch.sh` | start/stop/status/tail wrapper |
| `events.jsonl` | Event queue (created at runtime) |
| `state.json` | Persistent state — dedup, IMAP position (created at runtime) |

## Setup

1. Create a private Gmail account for the bridge
2. Enable 2FA on the account
3. Generate App Password: myaccount.google.com/apppasswords
4. Edit `~/.claude/email-bridge/config.json`:
   - `email`: the bridge Gmail address
   - `app_password`: the generated app password
   - `notify_to`: your personal email (where you receive notifications)
5. Start: `~/.claude/email-bridge/launch.sh start`

## Hooks

Configured in `~/.claude/settings.json` under the `hooks` key:
- `PermissionRequest` — fires when Claude needs tool approval
- `Notification` — fires on idle prompts, auth events
- `Stop` — fires when Claude stops working

## Usage

### Receiving notifications
- Emails arrive with subject like `[Claude] brain:0 — Allow Bash: git status?`
- Body contains session info, tool details, reply instructions

### Replying
- **Reply to a notification**: `y` to approve, `n` to deny, or any text as input
- **Direct command**: send email with subject `claude:<session>:<window> <command>`

### Daemon management
```bash
~/.claude/email-bridge/launch.sh start    # start daemon
~/.claude/email-bridge/launch.sh stop     # stop daemon
~/.claude/email-bridge/launch.sh status   # check status + recent logs
~/.claude/email-bridge/launch.sh tail     # follow logs
```

## Dedup Cooldowns

| Event | Cooldown |
|-------|----------|
| PermissionRequest | 0 (always send) |
| Notification | 5 min per session |
| Stop | 1 min per session |

## Implemented: 2026-03-23
