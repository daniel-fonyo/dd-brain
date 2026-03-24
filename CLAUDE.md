For any task, reference the repos below for code.
## Repos
- feed-service located in /Users/daniel.fonyo/Projects/feed-service
- services-protobuf located in /Users/daniel.fonyo/Projects/services-protobuf
	- Use this for proto contracts and schemas

## Project Management
You are in the following git project: /Users/daniel.fonyo/Projects/brain
This project is used to store all files and directories to be used by you for context.

### Brain Worktree Workflow (document changes)
1. **Start**: Create a worktree in `brain` at the very beginning of ANY interaction that requires creating or editing brain documents — no exceptions, even for small or quick edits.
2. **Work**: Make all document edits (plans, context, notes) inside the worktree branch.
3. **Merge**: When the task is complete (or whenever handing back to the user), merge the worktree branch back to `main` and remove the worktree. `main` is the source of truth — Obsidian reads from `main`. **Never leave commits only in a worktree branch when turning the conversation back to the user.**
4. Every document edit is committed so there is full version history of all context changes.

### Branch Verification (all repos)
Before making any changes in a referenced repo, **always verify the current branch** matches the intended target. Run `git branch --show-current` and confirm it aligns with the task. Never assume — check first.

### Referenced Repo Workflow (code changes)
For `feed-service`, `services-protobuf`, or any other referenced repo:
1. **Never merge to main directly.** Always work on a feature branch within that repo.
2. **Use worktrees for feed-service.** Multiple agents may work on this repo concurrently. Always create a worktree under `.claude/worktrees/<name>` for your feature branch instead of switching the main checkout. After creating the worktree, restore the main checkout to its original branch so other agents are not affected.
   ```bash
   cd /Users/daniel.fonyo/Projects/feed-service
   git branch my-feature  # create branch from current HEAD
   git worktree add .claude/worktrees/my-feature my-feature
   # Work inside .claude/worktrees/my-feature — leave the main checkout untouched
   ```
3. Commit with clear messages: what changed, why, and any testing notes.
4. Push the branch and open a PR with:
   - **Title**: concise summary
   - **Description**: what changed, why, how to test
5. **In brain documents**, reference the repo branch/PR so context is linked (e.g. `feed-service: feat/my-change` or PR URL).

### Permissions
- When the user grants a permission prompt during a session, immediately append the corresponding rule to the `allow` array in `~/.claude/settings.json` so it is pre-approved in future sessions.

### Personal Sandbox Test Setup (feed-service)
Before any sandbox deploy/test, apply these local-only changes (never commit):
1. In `HomepageRequestToContext.kt`, hardcode `getConsumerIdFromRequest` to `return 757606047L`
   - File: `pipelines/homepage/src/main/kotlin/com/doordash/consumer/pipelines/homepage/HomepageRequestToContext.kt`
2. **DANGER — Iguazu sandbox bypass** — In `IguazuModule.kt`, hardcode `val isSandboxEnv = false` to bypass sandbox Iguazu gating so events publish to Snowflake
   - File: `libraries/platform/src/main/kotlin/com/doordash/consumer/feed/platform/iguazu/IguazuModule.kt`
   - Without this, Iguazu events are blocked in sandbox by `enableSandboxIguazu()` and topic allowlist checks
   - **BE VERY CAREFUL**: This causes sandbox events to flow to PROD Iguazu/Snowflake. Only apply when testing specifically requires Snowflake validation. Revert immediately after testing. NEVER commit this change.

### Sandbox Browser Test Credentials
- **Domain**: `https://www.doordashtest.com/` (never `doordash.com`)
- **Email**: `tas-cx-doortest-egsn7vrtcl@doordash.com`
- **Password**: `1XQlCDr8Qb`
- Test account on doortest tenant. See `sandbox-e2e-testing/spec/00-browser-playbook.md` for full interaction sequence.

### Snowflake Access
- **Warehouse**: `ADHOC`
- **Database/Schema**: `IGUAZU.SERVER_EVENTS_PRODUCTION` (for Iguazu server events)
- **Auth**: `externalbrowser` (SSO via Okta — opens browser popup)
- **Connection config**: `~/.snowflake/connections.toml`
- **Python connector**: available at `/Users/daniel.fonyo/.local/share/uv/tools/mlf-tools/bin/python3` (has `snowflake.connector`)
- **MCP server**: configured in `~/.claude/mcp.json` as `snowflake` (requires session restart to activate)
- **CRITICAL**: Always filter by `iguazu_partition_date` AND `iguazu_partition_hour` — the tables are massive. Never scan more than 2 days without additional filters.

### feed-service: Detekt Pre-Commit Hook
The feed-service pre-commit hook runs `./gradlew detekt` which requires write access to `~/.gradle/wrapper/`. This **always fails** in the Claude Code sandbox because the sandbox blocks writes to `~/.gradle`. The hook output will show "Detekt failed" with empty "Detailed Output" sections — this is a sandbox restriction, not a real lint error.

**Workaround**: Use `git commit --no-verify` (with `dangerouslyDisableSandbox: true`) when committing in feed-service. The user should run detekt locally before pushing.

### Autonomy
- Do NOT use AskUserQuestion / elicitation prompts. Make your best judgment and proceed. If you're truly blocked with no reasonable default, state your assumption in a text message and continue.
- This is critical for remote workflows — elicitation prompts cannot be answered via the email bridge.

### Iguazu Validation
- Use **pod logging** (`kubectl logs` / `DEBUG_IGUAZU_*` tags) for sandbox event validation — NOT Snowflake.
- Snowflake is only for post-merge production verification when explicitly requested.

### General Rules
- Always make a plan before starting work. Execute the plan step by step.
- Always keep relevant brain documents up to date as you work. Create new documents when needed.
- All context from interactions must be captured and merged into brain. Keep documents concise — no fluff.
- Re-read documents after updating to verify they are tight and accurate.
- Every time I correct you, add a rule to the relevant CLAUDE.md so it never happens again.