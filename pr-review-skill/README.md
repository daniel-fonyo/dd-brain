# PR Review Skill (`/review-pr`)

Claude Code skill for structured, parallel, multi-dimensional PR review with human-in-the-loop submission.

**Not to be confused with `/feed-service-pr`** — that skill manages PRs you _author_ (create, push, status). This skill _reviews_ PRs you're added to as a reviewer.

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
1. **Gather** — fetches PR metadata, diff, existing comments (all parallel)
2. **Glean research + shape check** — searches Confluence for relevant docs, flags large/mixed PRs (parallel, after gather)
3. **Parallel dimension reviews** — up to 10 dimensions run as parallel subagents, informed by Glean context
4. **Collect & deduplicate** — parse JSON, validate line numbers against diff hunks, merge duplicates
5. **Post PENDING review** — inline diff comments only, never submitted
6. **Terminal output** — surfaces critical/bug findings prominently
7. **Self-learning** — updates skill files if new knowledge was discovered (after review is posted)

## Dimensions
| Dimension | File | Scope | What it checks |
|-----------|------|-------|----------------|
| correctness | `correctness.md` | All repos | Bugs, null safety, edge cases, races |
| design | `design.md` | All repos | Abstraction, coupling, scope creep |
| security | `security.md` | All repos | Injection, auth, data exposure |
| performance | `performance.md` | All repos | O(n²), N+1, blocking, unbounded growth |
| readability | `readability.md` | All repos | Naming, complexity, dead code |
| testing | `testing.md` | All repos | Coverage, fragile tests + feed-service mock/runTest rules |
| operability | `operability.md` | All repos | Logging, rollback, feature flags |
| kotlin-style | `kotlin-style.md` | `.kt` files | Google Android Kotlin Style Guide |
| feed-service | `feed-service.md` | doordash/feed-service | DV gating, coroutines, error handling, Double.NaN |
| pr-hygiene | `pr-hygiene.md` | doordash/feed-service | Required labels, testing evidence, description quality |

## Tags
All dimensions use the same 6 tags (unified vocabulary):

| Tag | Meaning |
|-----|---------|
| `[critical]` | Will crash/corrupt/security hole |
| `[bug]` | Logic error, race condition, missing DV gate |
| `[suggestion]` | Works but could be better |
| `[nit]` | Minor preference |
| `[super nit]` | Ultra-minor, take or leave |
| `[question]` | Asking, not asserting |

## Key Design Decisions
- **Line number validation** — findings are validated against diff hunks before posting; out-of-range lines are snapped to nearest changed line (prevents GitHub API rejection)
- **JSON parsing resilience** — subagent responses are parsed by finding `[`...`]` brackets (handles markdown fences or prose wrapping)
- **Glean graceful degradation** — if MCP is unavailable, review proceeds without internal doc context
- **DV gating ownership** — checked exclusively by `feed-service` dimension (no duplication across design/operability)
- **Zero findings** — if no issues found, prints encouragement and skips the API call (no empty review)

## References
- `references/google-android-kotlin-style-guide.md`
- `references/google-eng-practices-code-review.md`
- `references/feed-service-best-practices.md`
- `references/feed-service-unit-testing.md`
- `references/feed-service-pr-setup.md`

## Evolving the Skill
- **Automatic**: Self-learning updates skill files when Glean surfaces new rules
- **Add dimension**: create `dimensions/<name>.md`, add to SKILL.md list
- **Add style guide**: put in `references/`, create dimension gated on file extension
- **Add repo practices**: put in `references/`, create dimension gated on repo name
- **Tune tone**: edit Rules section and comment examples in SKILL.md
