# Presentation: Agentic Workflows for PR Quality

## Framing

The ask is "What is a high quality PR? What are blockers? How to remove them?" The real answer: **the blockers are human bandwidth and inconsistency, and the removal mechanism is agentic workflows that encode your standards into executable skills that improve with every run.**

This isn't a talk about AI doing code review. It's about building a system where the tedious, exhaustive, repetitive parts of PR quality are automated so humans can focus on judgment calls.

## Narrative Arc

### 1. The Problem (3 min)

What does a "high quality PR" actually mean? It's not one thing. It's 10+ dimensions checked simultaneously: correctness, design, security, performance, testing, operability, style, hygiene...

No human reviewer consistently checks all of them on every PR.

**The blockers:**

| Blocker | What happens |
|---------|-------------|
| **Reviewer bandwidth** | Engineers are bottlenecks for each other. Reviews sit for days. |
| **Inconsistency** | Different reviewers catch different things. Friday afternoon PRs get rubber-stamped. |
| **Knowledge fragmentation** | Best practices live in Confluence docs nobody reads. Kotlin style guide, unit testing efficiency, DV gating rules, Iguazu logging patterns, Double.NaN SEV-0 history... it's scattered across 15+ docs. |
| **Testing theater** | "Tested on sandbox" = loaded the page once, screenshot. No structured evidence, no cross-load comparison, no latency measurement, no log analysis. |
| **30-60 min manual validation** | A single homepage change takes 30-60 min of manual sandbox validation. This is the single biggest drag on velocity. |

### 2. The Thesis (1 min)

**Encode your team's standards into agentic skills. They run every time, miss nothing, and get smarter with use.**

Two systems built on Claude Code:
1. `/review-pr`: parallel multi-dimensional code review with self-verification
2. Autonomous sandbox testing agent: the engineer describes what to test, the agent handles how

Both are working today. Both found real bugs. Both produce better evidence than manual workflows.

### 3. Deep Dive: `/review-pr` (8 min)

#### The pipeline

```
PR URL
  |
  v
Phase 1: Gather (parallel: metadata, diff, files, existing comments)
  |
  v
Phase 2: Glean/Confluence Research + Shape Check
  |     Searches internal docs for applicable rules
  |     "events >1MB silently dropped", "Double.NaN caused SEV-0"
  |
  v
Phase 3: 10 Dimension Reviews (ALL parallel)
  |
  |  correctness  design  security  performance  readability
  |  testing  operability  kotlin-style  feed-service  pr-hygiene
  |
  v
Phase 3b: Self-Critic - Read Source Code
  |  Reads actual source to confirm/disprove uncertain findings
  |  Traces call chains, checks callers
  |
  v
Phase 3c: Self-Critic - Write Failing Test
  |  Creates worktree on PR branch
  |  Writes unit test for each confirmed bug
  |  Test FAILS = proven. Test PASSES = finding dropped.
  |
  v
Phase 4: Deduplicate, Validate, Post as PENDING
  |
  v
Phase 7: Self-Learning
  |  Updates its own skill files with new knowledge from Glean
```

#### Key points to land:

**Parallel, not serial.** 10 focused agents run simultaneously. A human reviewer is one person checking things sequentially, biased by what they know. This is 10 specialists running at once.

**Reads your Confluence.** The Glean integration searches internal docs before reviewing. It found "Iguazu events >1MB are silently dropped" and "Double.NaN in Protobuf serialization caused a SEV-0" from docs the reviewer might never have read.

**Doesn't just flag, verifies.** The self-critic loop is what makes this different from ChatGPT-paste-the-diff. It reads the actual source code to trace call chains. It writes a unit test to prove the bug exists. If the test passes, the finding is dropped. More rigorous than most human reviews.

**Sounds like a teammate.** Comments use first person, pose findings as questions, explain the mechanics, leave room for the author to correct. "I think `supervisorScope` suspends until all children complete? If so this `launch` blocks the return..." Not "This is wrong. Fix it."

**Human-in-the-loop.** Posts as PENDING review. You see every comment before submitting. The PR contains nothing you haven't reviewed.

**Gets smarter.** After each review, checks if Glean surfaced rules not yet in its knowledge base. Adds them. The skill compounds over time.

#### Real example: Bug found in PR #62639

The `/review-pr` skill found a real bug in [PR #62639](https://github.com/doordash/feed-service/pull/62639) that the author and other reviewers missed. Walk through:
1. What the skill flagged
2. How the verification agent confirmed it by reading the source
3. The failing test that proved it
4. How it would have been caught (or not) by a human reviewer

**This is the strongest demo moment.** A real bug, found by the skill, that would have shipped to production.

### 4. Deep Dive: Autonomous Sandbox Testing (8 min)

#### The problem (quote from RFC)

> "The homepage is a black box. Ranking inputs go in, a feed comes out, and verifying that the feed is correct requires correlating signals across multiple systems: service logs, correct rendering, Snowflake analytics events... A single change can take 30-60 minutes of manual validation."

#### The solution: describe what, agent handles how

The engineer sits down for 10-15 minutes, collaboratively builds a testing plan: what changed, what success looks like, what signals to check. Then the agent takes over.

#### The autonomous loop

```
Engineer's testing plan
  |
  v
Sync code to sandbox (continuous, async)
  |
  v
Drive real Chromium browser (Playwright)
  |  Navigate homepage, scroll, click carousels
  |  Enable debug overlays
  |
  v
Capture 5 independent signal sources (parallel):
  |
  |  Pod logs (kubectl)
  |  Structured extraction (carousel data, scores, latency)
  |  Screenshots + video recording
  |  Snowflake Iguazu events
  |  Remote debugging via injected debug logs
  |
  v
Analyze against success criteria
  |
  v
Decision: pass/fail/investigate further
  |  If something looks wrong, re-check
  |  If a carousel is missing, investigate why
  |
  v
Output: PR with full E2E evidence
  Session video, structured analysis, log snippets,
  Snowflake queries, pass/fail verdict with reasoning
```

#### Key points to land:

**Not a test runner. A testing agent.** It doesn't just execute a script. It reasons about what it sees. If scores look wrong, it re-checks. If a carousel is missing, it investigates why. This is the autonomous loop.

**Remote debugging without the debugger.** When it needs to "debug" like an engineer would in IntelliJ, it injects temporary debug logs into the code, syncs to sandbox, triggers loads, and parses the output. Same workflow, automated.

**3 independent loads.** Each load is fully isolated. Cross-load comparison detects non-determinism. Structured JSON extraction, not visual inspection.

**Self-healing.** UI changes break scripts. This agent adapts. If a button moves, a selector changes, or a page structure shifts, it has fallback strategies built in.

**Evidence that's actually useful.** Not "screenshot attached." Structured tables: per-load latency, carousel counts, store counts, error scan, stability analysis. Pod logs parsed for failures. Video recording of the full session.

#### Demo idea: Side-by-side

Show a real PR's Testing section:

**Before (manual):**
> "Tested on sandbox, homepage loads. Screenshot:"
> [one screenshot]

**After (agent):**
> | Load | Latency | Carousels | Stores | Request ID |
> |------|---------|-----------|--------|------------|
> | 1 | 1.2s | 12 | 47 | abc-123 |
> | 2 | 1.1s | 12 | 47 | def-456 |
> | 3 | 1.3s | 12 | 46 | ghi-789 |
>
> 11/12 carousels stable across all 3 loads. Pod errors: 0. Warnings: 2.
> Video: [link]. Full report: [link].

### 5. The Meta-Point: Skills as Encoded Standards (3 min)

#### Why this matters beyond "AI is cool"

1. **Tribal knowledge becomes executable.** The feed-service best practices doc, Kotlin style guide, unit testing efficiency guide, DV gating rules. They exist as docs. Nobody reads them consistently. Now they're checklists that run automatically.

2. **Consistency without fatigue.** Every PR gets the same 10-dimension review. Every sandbox test is 3 loads with structured evidence. Friday at 5pm gets the same quality as Monday at 10am.

3. **Compound learning.** Glean integration discovers new relevant docs. Self-learning adds new rules when patterns emerge. Each correction by the human makes the next run better. The flywheel spins.

4. **Fine-grained control.** Each dimension is a separate markdown file. Want to add a new check? Edit one file. Want to tune the tone? Edit the rules section. Want Go support? Create one new dimension. No code deployment, no CI pipeline. Just edit a text file.

5. **Human judgment where it matters.** The agent does the exhaustive scan (10 dimensions, Confluence search, source code verification, unit test writing). The human does the judgment call: "Is this finding worth blocking the PR?" That's the right division of labor.

#### The compound effect

> Each review makes the skill smarter. Each sandbox run improves the playbook. Each correction persists. In 6 months, the skill has absorbed every best practice, every post-mortem learning, every style preference from your entire team. That's not possible with human reviewers.

### 6. Demo (8-10 min)

#### Option A: Live review (high impact, higher risk)

1. Pick a real open PR (ideally medium complexity, mix of findings)
2. Run `/review-pr <url>` live in terminal
3. Show 10 parallel agents launching
4. Show Glean research finding relevant docs
5. Show PENDING review appearing on GitHub
6. Walk through 2-3 findings:
   - One `[suggestion]` with code organization feedback
   - One `[bug]` that was verified against source
   - One finding that was disproven and dropped (show the self-critic working)
7. Point to PR #62639 as the "found a real bug" moment

#### Option B: Walkthrough of saved results (lower risk)

1. Show the PR #62639 review where the skill found a real bug
2. Walk through the skill file tree: SKILL.md, dimensions/, references/
3. Open a dimension file, show how editable it is
4. Show the saved sandbox test evidence from a real `/feed-service-pr create` run
5. Show before/after PR descriptions

#### Option C: Hybrid (recommended)

1. Start with the PR #62639 bug story (saved results, no risk)
2. Open the skill files to show the architecture
3. Run a live review on a simple PR to show the parallel execution
4. Show the sandbox test evidence from a saved run

### 7. Q&A / Discussion

Seed questions:
- "What review checks does your team always miss?"
- "What tribal knowledge lives in Confluence that nobody reads?"
- "How long does sandbox testing take for your team's changes?"
- "What would you add as a new dimension for your team's repo?"

## Demo Prep Checklist

- [ ] PR #62639 review results ready to walk through (the bug find)
- [ ] One simple open PR ready for live review demo
- [ ] Terminal visible, zoomed to readable font size
- [ ] GitHub open in browser to show PENDING review appearing
- [ ] Pre-run one review to warm Glean auth and verify flow works
- [ ] Skill file tree ready to show (`~/.claude/skills/review-pr/`)
- [ ] One dimension file open to show editability (feed-service.md is best, most relatable)
- [ ] Saved sandbox test evidence (the structured table vs. "screenshot attached")
- [ ] RFC document open for reference during sandbox testing section
- [ ] Backup: pre-recorded terminal session in case live demo fails

## Slide Outline

1. **Title**: "Agentic Workflows for PR Quality"
2. **The problem**: what we miss, the 5 blockers table
3. **The thesis**: encode standards into skills
4. **Review pipeline**: architecture diagram
5. **10 dimensions**: the parallel breakdown
6. **Self-critic loop**: verify + test (the differentiator)
7. **Real bug found**: PR #62639 walkthrough
8. **Voice calibration**: Daniel's style vs AI slop (side-by-side)
9. **Sandbox testing**: the autonomous loop diagram
10. **Before/after evidence**: manual vs agent testing
11. **Skills = encoded standards**: the 5 meta-points
12. **Demo**

## Time Budget

| Section | Minutes |
|---------|---------|
| The problem + blockers | 3 |
| The thesis | 1 |
| /review-pr deep dive | 8 |
| Sandbox testing deep dive | 8 |
| Skills as encoded standards | 3 |
| Demo | 8-10 |
| Q&A | remaining |
| **Total** | ~30-35 min |
