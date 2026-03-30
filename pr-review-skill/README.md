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
1. **Gather** — fetches PR metadata, diff, existing comments (parallel)
2. **Shape check** — flags large/wide/mixed-concern PRs
3. **Parallel dimension reviews** — 8 dimensions run as parallel subagents
4. **Deduplicate** — merges findings, drops duplicates of existing feedback
5. **Post PENDING review** — comments go on the PR, never submitted (you submit manually)
6. **Terminal output** — surfaces critical/bug findings prominently

## Dimensions
| Dimension | File | What it checks |
|-----------|------|----------------|
| correctness | `dimensions/correctness.md` | Bugs, null safety, edge cases, races |
| design | `dimensions/design.md` | Abstraction, coupling, scope creep |
| security | `dimensions/security.md` | Injection, auth, data exposure |
| performance | `dimensions/performance.md` | O(n²), N+1, blocking, unbounded growth |
| readability | `dimensions/readability.md` | Naming, complexity, dead code |
| testing | `dimensions/testing.md` | Coverage gaps, fragile tests, assertions |
| operability | `dimensions/operability.md` | Logging, rollback, feature flags |
| kotlin-style | `dimensions/kotlin-style.md` | Google Android Kotlin Style Guide |

## Tags (used in comments)
- `[critical]` — will crash/corrupt/security hole
- `[bug]` — logic error, race condition
- `[suggestion]` — works but could be better
- `[nit]` — minor preference
- `[super nit]` — ultra-minor, take or leave
- `[question]` — asking, not asserting

## References
- `references/google-android-kotlin-style-guide.md` — Kotlin style authority
- `references/google-eng-practices-code-review.md` — Review philosophy

## Evolving the Skill
- Add new dimensions: create `dimensions/<name>.md` and add to the list in `SKILL.md`
- Add language style guides: put in `references/` and create a corresponding dimension
- Tune tone: edit the Rules section and comment examples in `SKILL.md`
- Add repo-specific context: could add `--context <file>` flag to inject codebase-specific knowledge
