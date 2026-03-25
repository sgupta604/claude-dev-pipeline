# Boreas Dev Pipeline

Claude Code development pipeline for boreas-web. Drop CLAUDE.md and .claude/ into the repo root.

## Setup

1. Copy `CLAUDE.md` and `.claude/` into the boreas repo root
2. Ensure `.claude/active-work/` is gitignored
3. Start Claude Code — it reads STATUS.md and prompts you

## Architecture

```
CLAUDE.md (150 lines, always in context)
  ↓ user types /research wind-api
.claude/commands/research.md (trigger, ~20 lines)
  ↓ orchestrator spawns
.claude/agents/research-agent.md (brain, 119 lines, fresh context)
  ↓ writes output to
.claude/features/wind-api/2026-03-24T22:00:00_research.md
```

Three layers: CLAUDE.md (always loaded) → Commands (triggers) → Agents (brains in isolated contexts).

## Commands

| Command | What |
|---------|------|
| `/research <feature>` | Gather requirements |
| `/plan <feature>` | Architecture + tasks |
| `/implement <feature>` | Build (delegates to specialists) |
| `/test <feature>` | Full suite + Playwright |
| `/finalize <feature>` | Commit, PR, retrospective |
| `/diagnose <feature>` | Root cause analysis |
| `/quickfix <desc>` | Small fix, no pipeline |
| `/hotfix <desc>` | Urgent fix, skip research |
| `/abort <feature>` | Revert broken implementation, stash changes |
| `/rework <feature>` | Archive approach, reset to research |
| `/park` | Pause current feature |
| `/resume <feature>` | Resume parked feature |
| `/status` | Show pipeline state |

## Agents

### Pipeline (orchestrate WHEN)
| Agent | Lines | Model | Purpose |
|-------|-------|-------|---------|
| research-agent | 119 | opus | Requirements + code analysis + retrospective review |
| plan-agent | 142 | opus | Architecture + task breakdown + contract-first streams |
| execute-agent | 135 | opus | Conductor — delegates to specialists, pre-flight checks |
| test-agent | 123 | sonnet | Run all suites, Playwright, handoff |
| finalize-agent | 122 | sonnet | Commit, PR, summary, retrospective, ADRs |
| diagnose-agent | 98 | opus | Root cause with evidence |

### Specialists (know HOW)
| Agent | Lines | Model | Domain |
|-------|-------|-------|--------|
| frontend-agent | 131 | opus | Next.js, Mapbox, Zustand, Tailwind, Vitest, Playwright |
| backend-agent | 156 | opus | FastAPI, SQLAlchemy, PostGIS, Redis, pytest (2-tier) |

## Key Design Decisions

- **Orchestrator dispatches, doesn't code** — handles pipeline state and config; delegates all `packages/` source code to agents
- **Agents run in fresh contexts** — immune to long-session degradation
- **Contract-first for cross-package features** — shared types defined in foundation stream before specialists begin
- **Specialists know Boreas** — Mapbox patterns, FastAPI conventions, wind API gotchas
- **3 tiers:** quickfix (trivial) / hotfix (urgent) / full pipeline (features)
- **Plan + tasks = 1 file** — 3 committed files per feature total
- **Error handling baked in** — max retries, BLOCKED marking, failure routing, `/abort` for recovery
- **Self-check on every agent** — verify before declaring done
- **Retrospective feedback loop** — research-agent reads past "Went Wrong" sections before starting
- **Session continuity** — session-log.md written before suggesting new sessions
- **ARCHITECTURE.md split** — all agents read it when CLAUDE.md outgrows 150 lines
- **Integration test tiers** — unit tests always run, integration tests need API keys
- **Diagnosis loop** — `/diagnose` feeds directly into `/plan` for targeted fix cycles
