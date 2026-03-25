---
description: Test a feature — spawns test-agent for full suite + Playwright + handoff checklist
---

Spawn the **test-agent** for feature: `$ARGUMENTS`

## Steps
1. Launch a test-agent: "Test feature '$ARGUMENTS'. Run pytest, Vitest, Playwright, lint, typecheck, build. Walk the handoff checklist in the plan doc. Write report to `.claude/active-work/$ARGUMENTS/`."
2. When agent returns:
   - **PASS:** Update STATUS.md: phase=`test-pass`, next=`/finalize $ARGUMENTS`. Suggest: "Tests passed! Run `/finalize $ARGUMENTS`?"
   - **FAIL:** Update STATUS.md: phase=`test-fail`, next=`/diagnose $ARGUMENTS`. Suggest: "Tests failed. Run `/diagnose $ARGUMENTS`?"
3. Give user the pass/fail summary with test counts

## Expected Output
**Pass:** `test-pass.md` with results table, build/lint status, handoff checklist verified.
**Fail:** `test-fail.md` with failures table, failure categories, and recommendation.

**Do NOT run tests yourself. Spawn the agent.**
