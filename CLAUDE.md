# Paperclip — Claude Code Project Guide

## What Is Paperclip

Paperclip is an **open-source orchestration platform for autonomous AI agent companies**. It's a Node.js/React control plane — not an agent framework itself — that coordinates teams of AI agents (OpenClaw, Claude Code, Codex, Cursor, HTTP webhooks, bash, etc.) running externally.

Key distinction: **OpenClaw is an employee, Paperclip is the company.** Agents heartbeat into Paperclip, check out tasks, do work, and report back.

- Multi-company, multi-agent, task-based coordination
- Budget enforcement, approval gates, audit logs, org charts
- Agents wake on cron schedule or manual trigger, maintain session state across runs
- Default port: **3100** (API + UI)

## Architecture

```
server/        Express REST API + orchestration engine
ui/            React + Vite board UI (served by server in dev)
cli/           paperclipai CLI (onboard, wake, doctor, worktree, etc.)
packages/
  db/          Drizzle ORM schema + migrations (PostgreSQL)
  shared/      TypeScript types, Zod validators, constants
  adapters/    Per-agent adapter packages
    claude-local/       Claude Code on local machine
    openclaw-gateway/   OpenClaw via WebSocket gateway
    codex-local/        Codex local
    cursor-local/       Cursor IDE
    gemini-local/       Gemini local
    opencode-local/     OpenCode local
    pi-local/           Pi model
  adapter-utils/  Shared adapter helpers
  plugins/        Plugin system packages
doc/           Specs, development guide, deployment docs
```

## Key Services

| File | Purpose |
|------|---------|
| `server/src/services/heartbeat.ts` | **Core**: Heartbeat execution loop (~147 KB) |
| `server/src/services/agents.ts` | Agent CRUD, status, config versioning |
| `server/src/services/routines.ts` | Cron-based scheduled jobs |
| `server/src/services/issues.ts` | Task/ticket lifecycle |
| `server/src/services/budgets.ts` | Budget enforcement + auto-pause |
| `server/src/services/costs.ts` | Token usage tracking |
| `server/src/services/cron.ts` | Cron expression parser |
| `server/src/index.ts` | Server startup + background scheduler |

## How Heartbeats Work

1. **Scheduler polls** every 5s — checks which agents are due to wake
2. **Context built** — agent config, assigned tasks, company skills, session history loaded
3. **Adapter invoked** — `execute()` called on appropriate adapter (HTTP, process, OpenClaw WS, etc.)
4. **Budget check** — hard-stop if monthly limit exceeded (agent auto-paused)
5. **Session persisted** — `agent_task_sessions` stores message history for next run
6. **Live events** — stdout/stderr streamed via WebSocket to UI in real-time
7. **Cost events** tracked — token usage attributed to agent + company

Session adapters (maintain cross-run state): `claude_local`, `codex_local`, `cursor`, `gemini_local`, `opencode_local`, `pi_local`

## How Agents Are Defined

Agents are generic — no special "charlie-001" builtin. Create via UI or CLI:

```json
{
  "name": "Charlie",
  "role": "CEO",
  "title": "Chief Executive Officer",
  "reportsTo": null,
  "capabilities": "Reviews strategy, assigns work, monitors spend",
  "adapterType": "openclaw_gateway",
  "adapterConfig": { "url": "ws://localhost:18789", "authToken": "..." },
  "runtimeConfig": { "workspaceRuntime": { "type": "git", "repo": "..." } },
  "budgetMonthlyCents": 500000
}
```

**Agent status enum:** `active | paused | idle | running | error | terminated`

If an agent's heartbeat stops firing, check:
1. Agent status — may be `paused` (budget exceeded or manual pause)
2. `last_heartbeat_at` in DB
3. `heartbeat_runs` table for error events
4. Scheduler running? — `PAPERCLIP_HEARTBEAT_SCHEDULER_ENABLED=true`

## OpenClaw / Mission Control Integration

OpenClaw connects via the **openclaw-gateway** adapter (`packages/adapters/openclaw-gateway/`):

- Paperclip opens a WebSocket to the OpenClaw gateway URL (e.g. `ws://localhost:18789`)
- Authenticates with device credentials (ED25519 keys) or token
- Sends `agent.run` request with task context, skills, budget constraints
- Receives streaming response frames (logs, cost events, completion)
- Device pair/approval flow for first-time connections

To wire up: create an agent with `adapterType: "openclaw_gateway"` and set the gateway URL in `adapterConfig`.

## Database

**Default (embedded PostgreSQL — zero config):**
```bash
# Data persists at ~/.paperclip/instances/default/db/
pnpm dev  # just works
```

**External PostgreSQL:**
```bash
export DATABASE_URL=postgres://user:pass@host:5432/db
pnpm dev
```

**Key tables:**
- `agents`, `agent_runtime_state`, `agent_task_sessions`, `agent_api_keys`
- `issues`, `issue_comments`, `issue_approvals`
- `heartbeat_runs`, `heartbeat_run_events`
- `routines`, `routine_runs`
- `cost_events`, `budget_policies`, `budget_incidents`
- `companies`, `projects`, `goals`
- `execution_workspaces`, `company_skills`, `company_secrets`

**Migrations:**
```bash
pnpm db:generate    # Create migration after schema change
pnpm db:migrate     # Apply pending migrations
pnpm db:backup      # Backup DB
```

## Known Issues / Gotchas

- **Heartbeat goes NULL**: Agent heartbeat can silently stop if the agent status becomes `paused` (budget limit hit) or if the scheduler restarts. Check `agent_runtime_state.status` and `heartbeat_runs` for recent errors.
- **External Postgres is password-protected**: When connecting to external Postgres, ensure the connection string includes credentials. Embedded PGlite doesn't require this.
- **API keys are company-scoped**: Agents authenticate with Bearer tokens scoped to their company. Cross-company access is intentionally blocked.
- **Session compaction**: Long-running agents have their session history compacted automatically. If an agent "forgets" context, check `agent_task_sessions.compact_threshold`.
- **Worktree instances**: Don't point two Paperclip servers at the same embedded DB. Use `paperclipai worktree init` to get an isolated instance.
- **pnpm-lock.yaml**: Do NOT commit this in PRs — CI owns it.

## Common Operations

```bash
# Start dev server (API + UI at localhost:3100)
pnpm dev             # watch mode
pnpm dev:once        # no file watching

# Stop/inspect managed dev runner
pnpm dev:list
pnpm dev:stop

# First-time setup
pnpm paperclipai onboard --yes
# or
pnpm paperclipai run   # auto-onboard + doctor + start

# Diagnose issues
pnpm paperclipai doctor

# Trigger agent heartbeat manually
pnpm paperclipai agent wake <agent-id>
# or via API: POST /api/agents/:id/wake

# Check agent status
# GET /api/agents/:id   (or view in UI at localhost:3100)

# Reset agent session (clear stuck state)
pnpm paperclipai agent reset-session <agent-id>
# or via API: POST /api/agents/:id/reset-session

# Run a routine immediately
pnpm paperclipai routines run <routine-id>
# or via API: POST /api/routines/:id/run

# Typecheck + test before merging
pnpm typecheck
pnpm test:run

# Build all packages
pnpm build
```

## API Endpoints (Key)

```
GET  /api/health
GET  /api/companies/:id/agents
POST /api/agents/:id/wake              # Trigger heartbeat
POST /api/agents/:id/reset-session     # Clear session
GET  /api/agents/:id/budget
POST /api/companies/:id/issues         # Create task
POST /api/issues/:id/checkout          # Atomic task assignment
POST /api/routines/:id/run             # Run routine now
GET  /api/companies/:id/costs          # Cost events
```

**Auth:** Board users use session-based auth (better-auth). Agents use `Bearer <api-key>` tokens. In `local_trusted` mode, board auth is implicit.

## Worktree Development

This repo uses git worktrees heavily for PR isolation:

```bash
pnpm paperclipai worktree:make <name>  # Create isolated worktree + DB instance
paperclipai worktree init               # Initialize worktree config in existing worktree
```

Each worktree gets its own Paperclip instance at `~/.paperclip-worktrees/instances/<worktree-id>/` with isolated DB. Routines are auto-paused in worktree instances.

## Key Documentation

- `doc/DEVELOPING.md` — Development setup, worktrees, storage, DB
- `doc/SPEC-implementation.md` — V1 build contract (26.7 KB, authoritative)
- `doc/DATABASE.md` — DB configuration options
- `doc/OPENCLAW_ONBOARDING.md` — OpenClaw integration guide
- `doc/DEPLOYMENT-MODES.md` — local_trusted vs authenticated modes
- `AGENTS.md` — Contribution rules, control-plane invariants, definition of done

## Tech Stack

- **Runtime:** Node.js 20+, pnpm 9+
- **Backend:** Express, TypeScript, Drizzle ORM, PostgreSQL (embedded PGlite or external)
- **Frontend:** React, Vite, TypeScript
- **Auth:** better-auth
- **Testing:** Vitest (unit), Playwright (E2E)
- **Monorepo:** pnpm workspaces
