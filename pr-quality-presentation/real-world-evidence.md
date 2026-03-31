# Real-World Evidence for Presentation

Numbers and stories to make the case concrete. Use these throughout the talk, not in a separate "metrics" section.

## Time Savings: Sandbox E2E Testing

| Metric | Manual | Agent | Delta |
|--------|--------|-------|-------|
| Single sandbox validation cycle | 30-60 min hands-on | 30-45 min background | Engineer is free the entire time |
| Evidence quality | 1 screenshot, no structure | 3 loads, structured tables, video, pod logs, Snowflake | Incomparably richer |
| Testing plan creation | lives in engineer's head | 10-15 min collaborative doc | Reusable, shareable |
| Non-determinism detection | impossible with 1 load | automatic with 3 loads | catches flaky ranking |

**Story to tell**: "I used to spend 30-60 minutes staring at a sandbox homepage, clicking through carousels, tailing pod logs, running Snowflake queries. Now I describe what I want to test, hand it off, and come back to a structured report. I've done this 3-4 times on real PRs. Each time the agent got better because I corrected it and those corrections persisted."

**The compound effect**: "The first run took some hand-holding. By the third run, I barely touched it. By the fifth run, someone else on my team could use the same playbook without any setup."

## Time Savings: PR Review

| Metric | Manual | Agent | Delta |
|--------|--------|-------|-------|
| Review start time | Hours to days (waiting for reviewer) | Immediate on invocation | No queue |
| Dimensions checked | 2-3 (what the reviewer knows) | 10 (every time) | 3-5x coverage |
| Time per review | 15-45 min focused human time | 2-3 min agent runtime | Human reviews the findings, not the diff |
| Glean/Confluence lookup | rarely done mid-review | automatic, every review | Catches tribal knowledge |
| Consistency | varies by reviewer and day | identical every time | no Friday rubber stamps |

**Story to tell**: "The review runs in 2-3 minutes. I spend another 5 minutes reading the findings before submitting. Total: ~8 minutes vs. 30+ minutes of reading every line of a 500-line diff myself. And I get 10 dimensions checked instead of the 2-3 I would have focused on."

## Bugs Found

### PR #62639: Real bug caught by `/review-pr`
- The skill flagged a correctness issue that the author and other reviewers missed
- Verified against source code (Phase 3b)
- This would have shipped to production without the skill
- **Tell the story**: what the PR changed, what the skill found, how it verified, what would have happened

### PR #59998 (own PR, used as test): Naming inconsistency + test gaps
- 5 suggestions, 1 question, 2 nits
- Naming inconsistency across 3 layers (`EXPECTED_VP_SCORE` vs `expectedCommission` vs `expectedVpPredictions`)
- 3 test coverage gaps (StoreCarouselDataAdapter, ContainerEventsGenerator, EntityRankerConfiguration store path)
- The naming issue alone would cause confusion for the next engineer touching Iguazu logging

### PR #62517: Clean PR, minimal findings
- Shows the skill doesn't over-flag. 3 suggestions on a well-scoped setup PR.
- "When the code is good, the skill says so. It's not a complaint machine."

## Quality of Feedback

**Side-by-side: human reviewer vs skill**

Human reviewer (common pattern):
> "LGTM"
> or
> "Looks good, one small nit: rename X to Y"

Skill output:
> "[bug] I believe if the NV Sibyl call throws (timeout, network error), `nvDeferred.await()` propagates the exception out of supervisorScope, failing the entire scoring request even though RX scoring already succeeded. supervisorScope prevents child cancellation during execution but does not swallow exceptions at await time. Do we need to wrap `nvDeferred.await()` in a try/catch that falls back to empty ScoresMetadata? Or is the exception handled further up the call stack?"

The skill explains the mechanics, shows reasoning, and asks if the fix is right. Most human reviews don't go this deep.

## Self-Learning in Action

**Glean discovery example from PR #59998 review:**
- Glean found the Iguazu Event Onboarding doc with rules like "events >1MB silently dropped" and "proto version must be >= services-protobuf tag"
- Found the Store Ranker Feed Service Details doc with the full scoring pipeline pattern
- Found the Double.NaN production alert doc
- These rules were passed to every dimension agent, improving the review quality

**Correction persistence:**
- After the first run, found that GitHub API rejects `"event": "PENDING"`. Fixed the skill.
- After the first run, found that line numbers outside diff hunks crash the API call. Added validation + snapping.
- After user feedback, added em dash ban, Daniel's voice style, code organization checks.
- Each of these persists forever. The skill never makes the same mistake twice.

## Framing for the Audience

Don't present this as "I built a cool AI thing." Present it as:

1. **"I was spending 30-60 min per sandbox test. Now I spend 10 min describing the plan and the agent runs in the background."** (time saved)
2. **"The skill found a bug in PR #62639 that everyone missed."** (quality improvement)
3. **"Every PR now gets 10-dimension review instead of whatever the reviewer happens to think about."** (consistency)
4. **"The skill reads our Confluence docs that we all know exist but nobody checks mid-review."** (tribal knowledge activation)
5. **"Each correction I make persists. The skill never makes the same mistake twice."** (compound learning)

## One-Liner Takeaways (for slides)

- "A high quality PR is one that's been checked across 10 dimensions, not just the 2-3 the reviewer happened to think about."
- "The blocker isn't lazy engineers. It's that thoroughness doesn't scale with human bandwidth."
- "The removal mechanism isn't better process docs. It's executable skills that encode the process and run it automatically."
- "This skill found a real bug that would have shipped to production. That's not a demo. That's value."
- "30-60 minutes of manual sandbox testing, replaced by 10 minutes of plan writing + background execution. The engineer is free the entire time."
