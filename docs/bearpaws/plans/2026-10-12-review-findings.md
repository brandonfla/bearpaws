# Review Findings Are Part of the Fix — Design and Plan

Date: 2026-10-12
Status: Approved (user chose this after D stopped)
Branch: `feat/review-findings`, based on `feat/routine-path`

## Problem (RED, from the pre-registered escaped-defect run)

In `docs/benchmarks/2026-10-12-escaped-defects.md`, Bearpaws let 9 of 30 defects escape. In at least 3 of them the independent reviewer flagged the exact defect, and the agent still shipped `Status: COMPLETE`. Two of the agents' final texts:

- "flagged the unfixed `sort` injection as a real, separate hole … Status: COMPLETE for the apostrophe fix."
- "non-admin cross-org owner access flagged, deliberately out of scope … Status: COMPLETE"

Rationalization: the bootstrap's elevated-risk rule asks for "a diff limited to the fix, with no unrelated cleanup". The agent reads a same-risk defect in the code it touched as unrelated.

## Change

Add one sentence to the bootstrap's `## Elevated risk` paragraph, directly after "Saying you skipped a step does not satisfy it.":

> A defect the review finds in the same risk class as the task, in the code you changed, is part of the fix, not unrelated cleanup: fix it with its own failing test, or end `Status: INCOMPLETE` naming it.

No other text changes. The narrow-diff rule stays for truly unrelated cleanup.

## Evaluation

- **GREEN:** `tests/benchmark/run.sh --suite risk --arms bearpaws --runs 5`, using the frozen C scenarios (about $7). Compare escaped, disclosed, and incomplete counts with C's Bearpaws arm (9 escaped, 1 disclosed, 5 incomplete). Success means fewer escapes, with reviewer-flagged defects now fixed or disclosed.
- **Regressions:**
  - `PLUGIN_DIR=$PWD tests/risk-eval/run.sh skills/using-bearpaws/SKILL.md after 3` must keep `narrow_diff` at 3/3. Its authz task has an unrelated `format_user` TODO that must stay untouched.
  - Routing eval: 5/5 cases.
  - Bootstrap growth under 250 bytes.
- **Comparison note:** this is a second run against the same frozen scenarios, so it measures the change in Bearpaws' own behavior. The C result stays the pre-registered cross-arm comparison.
