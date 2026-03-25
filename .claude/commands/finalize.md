---
description: Finalize a feature — spawns finalize-agent for commit, PR, summary with retrospective
---

Spawn the **finalize-agent** for feature: `$ARGUMENTS`

## Prerequisites
Verify `.claude/active-work/$ARGUMENTS/test-pass.md` exists. If not → "Run `/test $ARGUMENTS` first."

## Steps
1. Launch a finalize-agent: "Finalize feature '$ARGUMENTS'. Security sweep, write SUMMARY.md with retrospective, commit, create PR, update STATUS.md, clean active-work."
2. When agent returns, report the PR URL and summary to user

## Expected Output
The agent creates `SUMMARY.md` in features/, conventional commit, PR, updates STATUS.md, cleans active-work/.

**Do NOT finalize it yourself. Spawn the agent.**
