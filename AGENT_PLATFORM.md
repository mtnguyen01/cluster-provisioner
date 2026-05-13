# OpenClaw Agent Management Platform — Architecture Plan

**Goal:** A production Go service for managing fleets of OpenClaw agents.
**Backbone:** PostgreSQL + RabbitMQ
**Date:** 2026-05-13

---

## 1. Problem Statement

Engineers run multiple OpenClaw instances but lack:
- Centralized agent lifecycle management
- Scoped permissions per agent
- Health monitoring and auto-restart
- Audit trails for every agent action
- Git-managed configuration
- Approval gates for dangerous operations

**This platform is the control plane for OpenClaw agents.**

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│              OpenClaw Agent Management Platform                     │
│                    (Go, multi-replica capable)                      │
└─────────────────────────────────────────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
       ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
       │  PostgreSQL  │ │   RabbitMQ   │ │    Vault     │
       │   (state)    │ │   (events)   │ │  (secrets)   │
       └──────────────┘ └──────────────┘ └──────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Gateway    │      │   Agent 1    │      │   Agent N    │
│  (Control)   │      │  (ops-agent) │      │(support-agent│
└──────────────┘      └──────────────┘      └──────────────┘
```

---

## 3. Core Principles

| Principle | Implementation |
|-----------|---------------|
| **One Gateway, many agents** | Gateway routes traffic to named agents |
| **Separation by responsibility** | ops-agent, mail-agent, calendar-agent |
| **Scoped permissions** | Each agent only has tools it needs |
| **Health checks & restarts** | Docker/systemd/K8s with auto-restart |
| **Reproducible state** | Config in Git |
| **Approval gates** | Dangerous actions require approval |
| **Full observability** | Logs, metrics, audit trails |

---

## 4. Data Model

### agents

```sql
CREATE TABLE agents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE,
    display_name TEXT,
    description TEXT,
    role TEXT NOT NULL,
    team TEXT,
    config JSONB NOT NULL,
    bindings JSONB DEFAULT '[]',
    allowed_tools TEXT[] DEFAULT '{}',
    denied_tools TEXT[] DEFAULT '{}',
    require_approval_for TEXT[] DEFAULT '{"exec", "message", "gateway"}',
    status TEXT NOT NULL DEFAULT 'inactive',
    last_heartbeat TIMESTAMPTZ,
    last_error TEXT,
    workspace_path TEXT,
    pid INT,
    version TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);
```

### agent_runs

```sql
CREATE TABLE agent_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL REFERENCES agents(id),
    trigger_type TEXT NOT NULL,
    trigger_source TEXT,
    trigger_user TEXT,
    model TEXT,
    session_key TEXT,
    tools_called JSONB DEFAULT '[]',
    status TEXT NOT NULL,
    output TEXT,
    error_message TEXT,
    prompt_tokens INT,
    completion_tokens INT,
    cost_usd DECIMAL(10,6),
    duration_ms INT,
    approval_required BOOLEAN DEFAULT false,
    approved_by TEXT,
    approved_at TIMESTAMPTZ,
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
```

### approvals

```sql
CREATE TABLE approvals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id UUID NOT NULL REFERENCES agent_runs(id),
    agent_id UUID NOT NULL REFERENCES agents(id),
    action_type TEXT NOT NULL,
    action_details JSONB NOT NULL,
    requested_by TEXT NOT NULL,
    requested_at TIMESTAMPTZ DEFAULT NOW(),
    approved_by TEXT,
    approved_at TIMESTAMPTZ,
    rejection_reason TEXT,
    status TEXT NOT NULL DEFAULT 'pending',
    expires_at TIMESTAMPTZ
);
```

---

## 5. Service Components

### 5.1 Gateway

Routes incoming messages to the correct agent based on bindings.

```go
func (g *Gateway) Route(ctx context.Context, msg IncomingMessage) error {
    agentName := g.router.Resolve(msg.Channel, msg.Sender, msg.Content)
    agent, err := g.agentManager.Get(ctx, agentName)
    if err != nil { return err }
    if !agent.CanUseTool(msg.Intent) {
        return fmt.Errorf("agent %s cannot perform %s", agentName, msg.Intent)
    }
    return g.agentManager.Dispatch(ctx, agent, msg)
}
```

### 5.2 Agent Manager

Manages agent lifecycle: start, stop, restart, health checks.

```go
type Manager struct {
    store      AgentStore
    supervisor Supervisor
}

func (m *Manager) Start(ctx context.Context, agentID string) error {
    agent, _ := m.store.Get(ctx, agentID)
    m.ensureWorkspace(agent)
    m.writeConfig(agent)
    proc, _ := m.supervisor.Start(agent)
    agent.Status = "active"
    agent.PID = proc.PID
    return m.store.Update(ctx, agent)
}
```

### 5.3 Supervisor Backends

- Docker (docker run / compose)
- systemd (systemctl start/stop)
- Kubernetes (Deployment/StatefulSet)
- Local (os/exec)

### 5.4 Approval Service

```go
func (s *Service) RequestApproval(ctx context.Context, req ApprovalRequest) (*Approval, error) {
    approval := &Approval{...}
    s.store.Create(ctx, approval)
    s.notifier.Send(ctx, Notification{
        Channel: "slack",
        Target: "#ops-approvals",
        Message: fmt.Sprintf("Agent %s requests approval to %s", req.AgentName, req.ActionType),
        Actions: []Action{
            {Label: "Approve", URL: fmt.Sprintf("/approvals/%s/approve", approval.ID)},
            {Label: "Reject", URL: fmt.Sprintf("/approvals/%s/reject", approval.ID)},
        },
    })
    return approval, nil
}
```

---

## 6. Agent Lifecycle

```
[Inactive]
   │ POST /agents/:id/start
   ▼
[Starting] ──► [Error]
   │
   ▼
[Active] ◄──────┐
   │ Heartbeat   │
   │ timeout     │ Auto-restart
   ▼             │
[Unhealthy]     │
   └─────────────┘
   │ POST /agents/:id/stop
   ▼
[Inactive]
```

---

## 7. Permission Model

| Tool | ops-agent | mail-agent | calendar-agent |
|------|-----------|------------|----------------|
| web_search | ✅ | ✅ | ✅ |
| message | ✅ | ✅ | ❌ |
| read | ✅ | ✅ | ✅ |
| exec | ✅ | ❌ | ❌ |
| gateway | ✅ | ❌ | ❌ |
| cron | ✅ | ✅ | ✅ |
| nodes | ✅ | ❌ | ❌ |

---

## 8. Configuration as Code

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

---

## 9. API

```
GET    /v1/agents              → List agents
POST   /v1/agents              → Create agent
GET    /v1/agents/:id          → Get agent status
PATCH  /v1/agents/:id          → Update config
DELETE /v1/agents/:id          → Delete agent
POST   /v1/agents/:id/start    → Start agent
POST   /v1/agents/:id/stop     → Stop agent
POST   /v1/agents/:id/restart  → Restart agent
GET    /v1/agents/:id/logs     → Get logs
GET    /v1/agents/:id/runs     → Get run history

GET    /v1/runs                → List runs
GET    /v1/approvals           → List pending approvals
POST   /v1/approvals/:id/approve → Approve
POST   /v1/approvals/:id/reject  → Reject

GET    /v1/dashboard           → Overview metrics
```

---

## 10. Event Types (RabbitMQ)

```
agent.created
agent.started
agent.stopped
agent.restarted
agent.error
agent.heartbeat
run.started
run.completed
run.failed
run.approval_requested
approval.granted
approval.rejected
tool.called
config.changed
```

---

## 11. Docker Compose

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: openclaw
      POSTGRES_PASSWORD: openclaw
      POSTGRES_DB: openclaw_platform
    ports:
      - "5432:5432"

  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"

  vault:
    image: hashicorp/vault:latest
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: dev-token
    ports:
      - "8200:8200"

  platform:
    build: .
    ports:
      - "8080:8080"
    environment:
      DB_HOST: postgres
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672/
      VAULT_ADDR: http://vault:8200
      VAULT_TOKEN: dev-token

  ops-agent:
    image: openclaw/agent:latest
    environment:
      AGENT_NAME: ops-agent
      GATEWAY_URL: http://platform:8080
      ALLOWED_TOOLS: "web_search,message,read,exec,gateway,cron,nodes,image"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  mail-agent:
    image: openclaw/agent:latest
    environment:
      AGENT_NAME: mail-agent
      GATEWAY_URL: http://platform:8080
      ALLOWED_TOOLS: "web_search,message,read,cron"
      DENIED_TOOLS: "exec,gateway"
    restart: unless-stopped
```

---

## 12. MVP Scope

**Phase 1: Core**
- [ ] PostgreSQL schema
- [ ] Agent CRUD API
- [ ] Docker supervisor
- [ ] Start/stop/restart

**Phase 2: Gateway**
- [ ] Message routing
- [ ] Channel bindings

**Phase 3: Permissions**
- [ ] Tool allow/deny lists
- [ ] Approval gates
- [ ] Slack notifications

**Phase 4: Observability**
- [ ] Run audit logging
- [ ] Event bus
- [ ] Dashboard

**Phase 5: GitOps**
- [ ] Config as code
- [ ] Git webhook sync

**Phase 6: Production**
- [ ] Multi-replica
- [ ] K8s backend
- [ ] Cost tracking
