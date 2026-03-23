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
Before any sandbox deploy/test, apply this local-only change (never commit):
- In `HomepageRequestToContext.kt`, hardcode `getConsumerIdFromRequest` to `return 757606047L`
- File: `pipelines/homepage/src/main/kotlin/com/doordash/consumer/pipelines/homepage/HomepageRequestToContext.kt`

### Sandbox Browser Test Credentials
- **Domain**: `https://www.doordashtest.com/` (never `doordash.com`)
- **Email**: `tas-cx-doortest-egsn7vrtcl@doordash.com`
- **Password**: `1XQlCDr8Qb`
- Test account on doortest tenant. See `sandbox-e2e-testing/spec/00-browser-playbook.md` for full interaction sequence.

### General Rules
- Always make a plan before starting work. Execute the plan step by step.
- Always keep relevant brain documents up to date as you work. Create new documents when needed.
- All context from interactions must be captured and merged into brain. Keep documents concise — no fluff.
- Re-read documents after updating to verify they are tight and accurate.
- Every time I correct you, add a rule to the relevant CLAUDE.md so it never happens again.