# Email Bridge — Backlog

## In Progress

### Mobile-friendly email formatting
- All outbound emails should be HTML formatted for iPhone Gmail app
- Clean, scannable layout with proper font sizes and spacing
- Permission prompts: show tool name, input, and reply options prominently
- Stop events: show last assistant message truncated nicely
- Status digests: table of active sessions

### Mobile reply context injection
- When a reply arrives from email, inject a prefix before sending to Claude:
  ```
  [Replying from mobile — keep responses concise and mobile-readable.
   Format any output for narrow screen viewing.]
  ```
- This tells Claude the user is on their phone so it adapts its responses
- The injected text should be configurable in `config.json`

## Planned

### Outbound improvements
- [ ] Send HTML emails (not just plain text) for all notifications
- [ ] Include cmux workspace/tab name prominently in email body
- [ ] Truncate tool_input for long bash commands more aggressively
- [ ] Add "Reply 1/2/3" as bold options at top of email for quick scanning

### Inbound improvements
- [ ] Handle multi-line replies (currently only first line before quote is used)
- [ ] Support "!" prefix to send raw text without command mapping (e.g. "!y" sends literal "y")
- [ ] Queue commands if surface is busy (detect "waiting for response" state)

### Reliability
- [ ] Monitor Gmail rate limits — back off if throttled
- [ ] Alert (desktop notification via cmux) if IMAP connection drops for >60s
- [ ] Log rotation — bridge.log grows unbounded currently

### Future hooks
- [ ] Wire up `AskUserQuestion`/elicitation if Claude Code adds a hook for it
- [ ] Wire up `PreToolUse` / `PostToolUse` for selective tool monitoring
