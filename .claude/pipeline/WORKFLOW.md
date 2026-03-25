# Pipeline Workflow

## Flow

```
/research → /plan → /implement → /test → /finalize
                ^       ↑ /abort     ↓ (fail)
                +— /diagnose ←———————+
```

## Phases

### 1. Research → research-agent
**Writes:** `.claude/features/<feature>/YYYY-MM-DDTHH:MM:SS_research.md`

Six phases: gather context → extract requirements → analyze code → identify risks → resolve questions → recommend approach. Includes "Patterns to Follow" and "Code Examples from Spec" in output.

**Exit gate:** FRs/TRs listed, affected files identified, risks assessed, approach recommended.

### 2. Plan → plan-agent
**Writes:** `.claude/features/<feature>/YYYY-MM-DDTHH:MM:SS_plan.md` (includes tasks)

Architecture + task breakdown in ONE file. Tasks organized by streams with [P] markers. Each task has "Accepts when" criteria. Includes handoff checklist for test agent.

**Exit gate:** Architecture described, files listed, every task has acceptance criteria, non-goals stated.

### 3. Implement → execute-agent (delegates to frontend-agent / backend-agent)
**Writes:** `.claude/active-work/<feature>/progress.md`, checks off tasks in plan doc

Execute-agent is a **conductor**. It reads the plan, routes tasks by package:
- `packages/web/` tasks → **frontend-agent**
- `packages/api/` tasks → **backend-agent**
- `packages/shared/` → either, based on content
- Cross-cutting → execute-agent handles directly

**Error handling:** Max 2 retries per task. If still failing, mark BLOCKED, continue with non-dependent tasks.

**Exit gate:** All tasks checked off (or BLOCKED with documented reason), lint clean, tests pass.

### 4. Test → test-agent
**Writes:** `.claude/active-work/<feature>/test-pass.md` or `test-fail.md`

Runs: Vitest (frontend) + pytest (backend) + Playwright (E2E) + lint + build. Walks handoff checklist.

**Playwright rules:** Save failure screenshots to Playwright's default output. Reference by path. Don't embed. Don't clean up failures — they're evidence for /diagnose.

**Exit gate (pass):** All suites pass, build succeeds, analyzer clean, checklist verified.

### 5a. Finalize → finalize-agent
**Writes:** `.claude/features/<feature>/SUMMARY.md` (includes retrospective)

Cleanup → commit → PR → update STATUS.md → clean active-work/. Security checklist: no secrets, no debug code, no TODO hacks. Only 3 committed files per feature (research, plan, summary).

### 5b. Diagnose → diagnose-agent
**Writes:** `.claude/active-work/<feature>/diagnosis.md` (versioned: v2, v3...)

Root causes with evidence. Fix plan with effort estimates. Loops back to /plan.

## Alternative Paths

### Quick Fix (`/quickfix`)
< 3 files, obvious cause, low risk. Agent fixes + tests. No docs. If complex → recommends full pipeline.

### Hotfix (`/hotfix`)
Urgent production fix with clear requirements:
- **Skip** `/research` (requirements provided inline)
- **Abbreviated** `/plan` (can be a single task, inline in command)
- `/implement` → `/test` → `/finalize` still required
- Creates a hotfix branch: `hotfix/<name>`

### Park (`/park`)
Saves current state to STATUS.md. Active-work/ preserved. New feature can start.

### Resume (`/resume <feature>`)
Reads STATUS.md for last phase. Resumes from there. Re-reads docs (may be stale).

### Abort (`/abort`)
If `/implement` produced broken state and you want to start over:
1. `git stash` (preserves work, recoverable)
2. Cleans active-work/, unchecks tasks in plan
3. Resets to `plan-complete` — run `/implement` again or `/rework` for new approach

### Rework
Archives current plan as `*_abandoned.md`. Resets to /research. Old research kept.

## Infrastructure Rules

### Feature Names
Kebab-case, normalized. `user-auth` not `user-authentication`. The orchestrator normalizes before passing to agents.

### "Latest File" Resolution
Sort `*_plan.md` or `*_research.md` lexicographically (ISO 8601 timestamps with zero-padded hours). Last entry = latest.

### Branch Naming
`feat/<feature-name>`, `fix/<feature-name>`, `hotfix/<feature-name>`, `refactor/<feature-name>`.

### CLAUDE.md Growth Strategy
When CLAUDE.md exceeds 150 lines, move architecture details to `.claude/ARCHITECTURE.md` and reference it from CLAUDE.md. Keep CLAUDE.md focused on pipeline rules + essential project context.

### Commit Messages
`feat(web): add drift preview panel`
`fix(api): normalize Folkweather longitude`
`test(api): add wind endpoint integration tests`

## File Lifecycle

### Per Feature (committed) — 3 files max
```
.claude/features/<feature>/
  YYYY-MM-DDTHH:MM:SS_research.md
  YYYY-MM-DDTHH:MM:SS_plan.md
  SUMMARY.md
```

### Working Files (gitignored, cleaned after finalize)
```
.claude/active-work/<feature>/
  progress.md, test-pass.md, test-fail.md, diagnosis.md
```

## Worktree Safety
- Max 2 parallel worktrees
- Never branch from HEAD with uncommitted changes
- Use git merge, not file copying
- Run full tests after merge
- Parallel streams must not touch same files

## Future: Rust Agent (Phase 3)

`packages/physics/` is a three-crate Rust workspace (drift-core, drift-python/PyO3, drift-wasm). Neither frontend-agent nor backend-agent knows Rust. Before Phase 3 begins, create `.claude/agents/rust-agent.md` covering: nalgebra, wasm-bindgen, maturin builds, PyO3 bindings, floating-point determinism, and `cargo test`. Until then, the pipeline handles Phases 1-2 fully.

## Agent Output Rules
1. Concise — summaries under 500 words
2. Reference by path — don't paste code blocks
3. Signal over noise — only info that helps the next phase
4. Screenshots by path — never inline
5. Omit empty sections
6. Each agent writes ONE output file (plus checking off tasks)
