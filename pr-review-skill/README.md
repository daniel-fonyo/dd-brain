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
1. **Gather + Glean research** — fetches PR data AND searches Glean/Confluence for relevant docs (parallel)
2. **Shape check** — flags large/wide/mixed-concern PRs
3. **PR description & testing check** — validates labels, testing evidence, description quality
4. **Parallel dimension reviews** — up to 10 dimensions run as parallel subagents, informed by Glean context
5. **Deduplicate** — merges findings, drops duplicates of existing feedback
6. **Post PENDING review** — inline diff comments only (never top-level), never submitted
7. **Terminal output** — surfaces critical/bug findings prominently with file:line
8. **Self-learning** — if new knowledge was discovered, updates skill files for future reviews

## Dimensions
| Dimension | File | What it checks |
|-----------|------|----------------|
| correctness | `dimensions/correctness.md` | Bugs, null safety, edge cases, races |
| design | `dimensions/design.md` | Abstraction, coupling, DV gating, scope creep |
| security | `dimensions/security.md` | Injection, auth, data exposure |
| performance | `dimensions/performance.md` | O(n²), N+1, blocking, unbounded growth |
| readability | `dimensions/readability.md` | Naming, complexity, dead code |
| testing | `dimensions/testing.md` | Coverage, mock lifecycle, runTest, no copypasta |
| operability | `dimensions/operability.md` | Logging, rollback, feature flags |
| kotlin-style | `dimensions/kotlin-style.md` | Google Android Kotlin Style Guide |
| feed-service | `dimensions/feed-service.md` | DoorDash internal best practices + DV gating |
| pr-hygiene | `dimensions/pr-hygiene.md` | Labels, description, testing evidence |

## Glean Integration
The skill searches Glean/Confluence in parallel with data gathering to find:
- Relevant feed-service wiki pages
- Engineering best practices docs
- Library usage guides
- Design docs and RFCs
- Post-mortem learnings

Applicable rules from Glean docs are passed to every dimension subagent.

## Self-Learning
After each review, the skill checks if new knowledge should persist:
- **New Glean docs** with rules not yet in references → saved to `references/`
- **Missing checklist items** discovered during review → added to dimensions
- **Recurring patterns** across PRs → codified in dimensions
- Prints what was updated to the terminal

## Tags (used in inline comments)
- `[critical]` — will crash/corrupt/security hole
- `[bug]` — logic error, race condition, missing DV gate
- `[suggestion]` — works but could be better
- `[nit]` — minor preference
- `[super nit]` — ultra-minor, take or leave
- `[question]` — asking, not asserting

## Key Rules
- **NEVER submits the review** — always PENDING, you submit manually
- **ALL comments are inline diff comments** — no top-level, so you can approve
- **DV gating mandatory** — new features/behavior changes without DV = `[bug]`
- **Human tone** — no AI slop
- **Self-evolving** — learns from each review

## References
- `references/google-android-kotlin-style-guide.md` — Kotlin style authority
- `references/google-eng-practices-code-review.md` — Review philosophy
- `references/feed-service-best-practices.md` — Internal coding standards
- `references/feed-service-unit-testing.md` — Efficient unit testing guide
- `references/feed-service-pr-setup.md` — Label requirements & CI pipeline

## Evolving the Skill
- **Automatic**: The skill updates itself when Glean surfaces new applicable rules
- **Manual — Add dimension**: create `dimensions/<name>.md`, add to SKILL.md list
- **Manual — Add style guide**: put in `references/`, create a corresponding dimension
- **Manual — Add repo practices**: put in `references/`, create a dimension gated on repo name
- **Manual — Tune tone**: edit Rules section and comment examples in SKILL.md
