# Boreas Dev Pipeline

Claude Code development pipeline for boreas-web. Drop CLAUDE.md and .claude/ into the repo root.

## Setup

1. Copy `CLAUDE.md` and `.claude/` into the boreas repo root
2. Ensure `.claude/active-work/` is gitignored
3. Start Claude Code — it reads STATUS.md and prompts you

## Architecture

```
CLAUDE.md (135 lines, always in context)
  ↓ user types /research wind-api
.claude/commands/research.md (trigger, 20 lines)
  ↓ orchestrator spawns
.claude/agents/research-agent.md (brain, 118 lines, fresh context)
  ↓ writes output to
.claude/features/wind-api/2026-03-24T22:00_research.md
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
| `/park` | Pause current feature |
| `/resume <feature>` | Resume parked feature |
| `/status` | Show pipeline state |

## Agents

### Pipeline (orchestrate WHEN)
| Agent | Lines | Model | Purpose |
|-------|-------|-------|---------|
| research-agent | 118 | opus | Requirements + code analysis |
| plan-agent | 129 | opus | Architecture + task breakdown |
| execute-agent | 123 | opus | Conductor — delegates to specialists |
| test-agent | 115 | sonnet | Run all suites, Playwright, handoff |
| finalize-agent | 119 | sonnet | Commit, PR, summary, retrospective |
| diagnose-agent | 98 | opus | Root cause with evidence |

### Specialists (know HOW)
| Agent | Lines | Model | Domain |
|-------|-------|-------|--------|
| frontend-agent | 121 | opus | Next.js, Mapbox, Zustand, Tailwind, Vitest |
| backend-agent | 144 | opus | FastAPI, SQLAlchemy, PostGIS, Redis, pytest |

## Key Design Decisions

- **Orchestrator never works** — dispatches only
- **Agents run in fresh contexts** — immune to long-session degradation
- **Specialists know Boreas** — Mapbox patterns, FastAPI conventions, wind API gotchas
- **3 tiers:** quickfix (trivial) / hotfix (urgent) / full pipeline (features)
- **Plan + tasks = 1 file** — 3 committed files per feature total
- **Error handling baked in** — max retries, BLOCKED marking, failure routing
- **Self-check on every agent** — verify before declaring done
