# Email Bridge for Claude Code cmux Sessions

Sends email notifications from Claude Code hooks and accepts commands via email replies, routed to cmux surfaces.

## Architecture

```
Claude Code hooks → events.jsonl → bridge.py daemon → Gmail (SMTP)
Gmail (IMAP) → bridge.py → cmux send → Claude session
```

## Files

All live in `~/.claude/email-bridge/`:

| File | Purpose |
|------|---------|
| `config.json` | Gmail creds, cmux socket password, poll intervals |
| `hook_notify.py` | Hook script — appends events to queue with cmux surface refs |
| `bridge.py` | Main daemon — email send/receive + cmux routing |
| `launch.sh` | start/stop/status/tail wrapper (also used for manual restarts) |
| `events.jsonl` | Event queue (created at runtime) |
| `state.json` | Persistent state — dedup, IMAP position, notification map |

## Auto-Start

LaunchAgent: `~/Library/LaunchAgents/com.claude.email-bridge.plist`
- Starts on login, restarts on crash (`KeepAlive: true`)
- `launchctl stop com.claude.email-bridge` to restart
- `launchctl unload ~/Library/LaunchAgents/com.claude.email-bridge.plist` to disable

## Setup

1. Private Gmail: `danie.fonyo.claude.agent@gmail.com`
2. App Password from myaccount.google.com/apppasswords
3. Notify to: `daniel.fonyo@doordash.com`
4. cmux socket mode: `cmuxOnly` with password auth

## Hooks (in `~/.claude/settings.json`)

- `PermissionRequest` — fires when Claude needs tool approval
- `Notification` — fires on idle prompts, auth events
- `Stop` — fires when Claude stops working
- **NOT hookable**: `AskUserQuestion` (elicitation prompts) — must be answered from Mac

**Important**: Existing sessions don't pick up new hooks. Restart Claude in each tab after adding/changing hooks.

## Reply Commands

| You reply | Bridge sends | Effect |
|-----------|-------------|--------|
| `y` | `1` | Yes (approve permission) |
| `ya` | `2` | Yes, allow for session |
| `n` | `3` | No (deny) |
| `1`-`4` | as-is | Select option by number |
| any text | as-is | Typed as input |

## Email Subjects

Format: `[Claude] <workspace>/<tab> — <event summary>`

Example: `[Claude] Embedding Score Logging/Sandbox Testing — Allow Bash: git status`

Workspace and tab names come from cmux tree, resolved via surface refs captured by the hook.

## Dedup Cooldowns

| Event | Cooldown |
|-------|----------|
| PermissionRequest | 0 (always send) |
| Notification | 5 min per session |
| Stop | 1 min per session |

## Known Limitations

- Emails land in spam initially — mark "not spam" a few times, Gmail learns
- `AskUserQuestion` prompts can't be emailed (no hook event for them)
- Sessions running before hooks were added need restart or manual notification
- cmux socket requires `allowAll` or password auth for daemon access

## Implemented: 2026-03-23
