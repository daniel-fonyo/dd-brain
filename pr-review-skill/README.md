# PR Review Skill

Claude Code skill for structured, parallel, multi-dimensional PR review with human-in-the-loop submission.

## Location
`~/.claude/skills/review-pr/` (personal — works across all repos)

## Usage
```
/review-pr doordash/feed-service#1234
/review-pr https://github.com/doordash/feed-service/pull/1234
/review-pr 1234                          # infers repo from CWD
/review-pr 1234 --focus correctness,security
/review-pr 1234 --severity-only bug
```

## How It Works
1. **Gather** — fetches PR metadata, diff, existing comments (all in parallel)
2. **Shape check** — flags large/wide/mixed-concern PRs
3. **PR description & testing check** — validates labels, testing evidence, description quality
4. **Parallel dimension reviews** — up to 10 dimensions run as parallel subagents
5. **Deduplicate** — merges findings, drops duplicates of existing feedback
6. **Post PENDING review** — inline diff comments only (never top-level), never submitted
7. **Terminal output** — surfaces critical/bug findings prominently with file:line

## Dimensions
| Dimension | File | What it checks |
|-----------|------|----------------|
| correctness | `dimensions/correctness.md` | Bugs, null safety, edge cases, races |
| design | `dimensions/design.md` | Abstraction, coupling, scope creep |
| security | `dimensions/security.md` | Injection, auth, data exposure |
| performance | `dimensions/performance.md` | O(n²), N+1, blocking, unbounded growth |
| readability | `dimensions/readability.md` | Naming, complexity, dead code |
| testing | `dimensions/testing.md` | Coverage, fragile tests, mock lifecycle, runTest |
| operability | `dimensions/operability.md` | Logging, rollback, feature flags |
| kotlin-style | `dimensions/kotlin-style.md` | Google Android Kotlin Style Guide |
| feed-service | `dimensions/feed-service.md` | DoorDash feed-service best practices |
| pr-hygiene | `dimensions/pr-hygiene.md` | Labels, description, testing evidence |

## Tags (used in inline comments)
- `[critical]` — will crash/corrupt/security hole
- `[bug]` — logic error, race condition
- `[suggestion]` — works but could be better (syntax, idiom, pattern)
- `[nit]` — minor preference
- `[super nit]` — ultra-minor, take or leave
- `[question]` — asking, not asserting

## References
- `references/google-android-kotlin-style-guide.md` — Kotlin style authority
- `references/google-eng-practices-code-review.md` — Review philosophy
- `references/feed-service-best-practices.md` — Internal coding standards
- `references/feed-service-unit-testing.md` — Efficient unit testing guide
- `references/feed-service-pr-setup.md` — Label requirements & CI pipeline

## Key Rules
- **NEVER submits the review** — always PENDING, you submit manually on GitHub
- **ALL comments are inline diff comments** — no top-level PR comments, so you can still approve
- **Human-sounding tone** — no AI slop, direct conversational comments
- **Breaking issues flagged in terminal** — critical/bug findings shown prominently before you go to GitHub
- **Google eng-practices philosophy** — approve if it improves code health, don't block for perfection

## Evolving the Skill
- **Add dimension**: create `dimensions/<name>.md`, add to SKILL.md list
- **Add style guide**: put in `references/`, create a dimension gated on file extension
- **Add repo-specific practices**: put in `references/`, create a dimension gated on repo name
- **Tune tone/tags**: edit Rules section and comment examples in SKILL.md
- **Add language**: create `dimensions/<lang>-style.md` with checklist from official style guide
