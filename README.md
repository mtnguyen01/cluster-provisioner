# OpenClaw Agent Management Platform

A production Go service for managing fleets of OpenClaw agents.

## Problem

Engineers run multiple OpenClaw instances but lack:
- Centralized agent lifecycle management
- Scoped permissions per agent
- Health monitoring and auto-restart
- Audit trails for every agent action
- Git-managed configuration
- Approval gates for dangerous operations

## Solution

This platform is the **control plane for OpenClaw agents**:
- One Gateway routes traffic to named agents
- Each agent has scoped permissions (no shell for calendar agent)
- Health checks with auto-restart
- Full audit trail of every run
- Git-managed config (agents are "config + runtime state")
- Approval gates for dangerous actions

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              OpenClaw Agent Management Platform             │
│                   (Go, multi-replica)                       │
└─────────────────────────────────────────────────────────────┘
              │
    ┌─────────┴─────────┐
    │                   │
    ▼                   ▼
┌──────────┐    ┌──────────────┐
│ Postgres │    │   RabbitMQ   │
│  (state) │    │   (events)   │
└──────────┘    └──────────────┘
```

## Core Principles

1. **One Gateway, many agents** — Gateway routes traffic to named agents via bindings
2. **Separation by responsibility** — ops-agent, mail-agent, calendar-agent, support-agent
3. **Scoped permissions** — each agent only has tools it needs
4. **Health checks & restarts** — Docker/systemd/K8s with auto-restart
5. **Reproducible state** — Config in Git, not pets
6. **Approval gates** — Dangerous actions require human approval
7. **Full observability** — Logs, metrics, audit trails for every run

## Agent Types

| Agent | Role | Allowed Tools | Denied Tools |
|-------|------|--------------|--------------|
| **ops-agent** | Infrastructure | All | — |
| **mail-agent** | Email handling | web_search, message, read, cron | exec, gateway |
| **calendar-agent** | Scheduling | web_search, read, cron | exec, gateway, message |
| **support-agent** | Customer support | web_search, message, read | exec, gateway |
| **devops-agent** | CI/CD | web_search, exec, read, message | gateway |
| **finance-agent** | Read-only finance | web_search, read | exec, message, gateway |

## API

```
GET    /v1/agents              → List agents
POST   /v1/agents              → Create agent
GET    /v1/agents/:id          → Get agent status
POST   /v1/agents/:id/start    → Start agent
POST   /v1/agents/:id/stop     → Stop agent
POST   /v1/agents/:id/restart  → Restart agent
GET    /v1/agents/:id/logs     → Get agent logs
GET    /v1/agents/:id/runs     → Get run history

GET    /v1/runs                → List runs (audit trail)
GET    /v1/approvals           → List pending approvals
POST   /v1/approvals/:id/approve → Approve action

GET    /v1/dashboard           → Overview metrics
```

## Quick Start

```bash
# Start dependencies
docker compose up -d

# Run migrations
make migrate-up

# Create an agent
curl -X POST http://localhost:8080/v1/agents \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ops-agent",
    "role": "ops",
    "allowed_tools": ["web_search", "message", "read", "exec", "gateway"],
    "bindings": [{"channel": "telegram", "target": "@ops_bot"}]
  }'

# Start the agent
curl -X POST http://localhost:8080/v1/agents/ops-agent/start

# Check status
curl http://localhost:8080/v1/agents/ops-agent
```

## Configuration as Code

```
config/
├── agents/
│   ├── ops-agent/
│   │   ├── agent.json
│   │   ├── SOUL.md
│   │   └── TOOLS.md
│   └── mail-agent/
│       └── ...
├── prompts/
│   └── system-prompts/
└── policies.yaml
```

Changes pushed to Git are automatically rolled out to agents.

## Status

🚧 **Architecture phase** — design complete, implementation starting.

## References

- OpenClaw: https://github.com/openclaw/openclaw
- Docs: https://docs.openclaw.ai
