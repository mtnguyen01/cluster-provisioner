# OpenClaw Agent Management Platform — Architecture Plan

**Goal:** A production Go service for managing a fleet of OpenClaw agents across teams and use cases.
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
       │   (state)    │ │  (events)    │ │  (secrets)   │
       └──────────────┘ └──────────────┘ └──────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Gateway    │      │   Agent 1    │      │   Agent N    │
│  (Control)   │      │  (ops-agent) │      │(support-agent│
└──────────────┘      └──────────────┘      └──────────────┘
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │  External Services  │
                    │  (Telegram, Discord,│
                    │   Slack, Email,     │
                    │   Calendar, etc.)   │
                    └─────────────────────┘
```

---

## 3. Core Principles

| Principle | Implementation |
|-----------|---------------|
| **One Gateway, many agents** | Gateway routes traffic to named agents via bindings |
| **Separation by responsibility** | Each agent has a specific role (ops, mail, calendar, support) |
| **Scoped permissions** | Each agent only has tools it needs — no shell for calendar agent |
| **Health checks & restarts** | Docker Compose / systemd / K8s with auto-restart |
| **Reproducible state** | Config in Git: agents are "config + runtime state," not pets |
| **Approval gates** | Dangerous actions require explicit approval |
| **Full observability** | Logs, metrics, audit trails for every run |

---

## 4. Data Model (PostgreSQL)

### 4.1 agents

```sql
CREATE TABLE agents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE,        -- "ops-agent", "mail-agent"
    display_name TEXT,                 -- "Operations Agent"
    description TEXT,
    
    -- Classification
    role TEXT NOT NULL,                -- "ops", "mail", "calendar", "support", "devops", "finance"
    team TEXT,                         -- "platform", "customer-success", "engineering"
    
    -- Configuration
    config JSONB NOT NULL,             -- agent definition, prompts, model settings
    bindings JSONB DEFAULT '[]',       -- channel bindings: [{"channel": "telegram", "target": "..."}]
    
    -- Permissions
    allowed_tools TEXT[] DEFAULT '{}', -- ["web_search", "message", "read"]
    denied_tools TEXT[] DEFAULT '{}',  -- ["exec", "gateway"]
    
    -- Approval gates
    require_approval_for TEXT[] DEFAULT '{"exec", "message", "gateway"}',
    
    -- State
    status TEXT NOT NULL DEFAULT 'inactive', -- inactive, active, error, paused
    last_heartbeat TIMESTAMPTZ,
    last_error TEXT,
    
    -- Runtime
    workspace_path TEXT,               -- /var/lib/openclaw/agents/ops-agent
    pid INT,                           -- process ID
    version TEXT,                      -- OpenClaw version
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ             -- soft delete
);

CREATE INDEX idx_agents_role ON agents(role);
CREATE INDEX idx_agents_status ON agents(status);
CREATE INDEX idx_agents_team ON agents(team);
```

### 4.2 agent_runs (audit trail)

```sql
CREATE TABLE agent_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL REFERENCES agents(id),
    
    -- Trigger
    trigger_type TEXT NOT NULL,        -- "message", "cron", "heartbeat", "manual"
    trigger_source TEXT,               -- "telegram:8756405644", "cron:backup-check"
    trigger_user TEXT,                 -- who triggered it
    
    -- Execution
    model TEXT,                        -- "kimi/kimi-code", "anthropic/claude-opus-4-7"
    session_key TEXT,                  -- OpenClaw session identifier
    
    -- Tools used
    tools_called JSONB DEFAULT '[]',   -- [{"tool": "exec", "args": "...", "result": "..."}]
    
    -- Result
    status TEXT NOT NULL,              -- running, completed, failed, approval_pending
    output TEXT,                       -- agent response
    error_message TEXT,
    
    -- Metrics
    prompt_tokens INT,
    completion_tokens INT,
    cost_usd DECIMAL(10,6),
    duration_ms INT,
    
    -- Approval
    approval_required BOOLEAN DEFAULT false,
    approved_by TEXT,
    approved_at TIMESTAMPTZ,
    
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_runs_agent_id ON agent_runs(agent_id);
CREATE INDEX idx_runs_status ON agent_runs(status);
CREATE INDEX idx_runs_trigger ON agent_runs(trigger_type, trigger_source);
CREATE INDEX idx_runs_started ON agent_runs(started_at);
```

### 4.3 agent_events

```sql
CREATE TABLE agent_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL REFERENCES agents(id),
    
    event_type TEXT NOT NULL,          -- heartbeat, started, stopped, error, tool_called, approved
    severity TEXT DEFAULT 'info',      -- debug, info, warning, error, critical
    
    payload JSONB,                     -- event-specific data
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_events_agent_id ON agent_events(agent_id);
CREATE INDEX idx_events_type ON agent_events(event_type);
CREATE INDEX idx_events_created ON agent_events(created_at);
```

### 4.4 agent_configs (Git-managed)

```sql
CREATE TABLE agent_configs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL REFERENCES agents(id),
    
    config_type TEXT NOT NULL,         -- "agent_json", "prompts", "tools", "bindings"
    content TEXT NOT NULL,             -- JSON/YAML content
    
    -- Git tracking
    git_commit TEXT,
    git_branch TEXT,
    synced_at TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 4.5 approvals

```sql
CREATE TABLE approvals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id UUID NOT NULL REFERENCES agent_runs(id),
    agent_id UUID NOT NULL REFERENCES agents(id),
    
    action_type TEXT NOT NULL,         -- "exec", "message", "gateway", "delete"
    action_details JSONB NOT NULL,     -- { "command": "rm -rf /", "reason": "cleanup" }
    
    requested_by TEXT NOT NULL,        -- agent name
    requested_at TIMESTAMPTZ DEFAULT NOW(),
    
    approved_by TEXT,
    approved_at TIMESTAMPTZ,
    rejection_reason TEXT,
    
    status TEXT NOT NULL DEFAULT 'pending', -- pending, approved, rejected, expired
    expires_at TIMESTAMPTZ,            -- auto-expire if not acted on
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_approvals_status ON approvals(status);
CREATE INDEX idx_approvals_run_id ON approvals(run_id);
CREATE INDEX idx_approvals_expires ON approvals(expires_at) WHERE status = 'pending';
```

### 4.6 channels (external service bindings)

```sql
CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE,         -- "telegram-prod", "discord-team"
    
    channel_type TEXT NOT NULL,        -- "telegram", "discord", "slack", "email"
    config JSONB NOT NULL,             -- { "token": "...", "webhook": "..." }
    
    -- Binding to agent
    agent_id UUID REFERENCES agents(id),
    binding_rules JSONB DEFAULT '{}',  -- routing rules
    
    status TEXT DEFAULT 'active',
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 5. Service Components

### 5.1 Gateway (Control Point)

The Gateway is the single entry point for all external traffic.

```go
package gateway

type Gateway struct {
    // Routing
    router *Router
    
    // Agent management
    agentManager *AgentManager
    
    // Channels
    channels map[string]Channel
    
    // Events
    eventBus EventBus
}

func (g *Gateway) Route(ctx context.Context, msg IncomingMessage) error {
    // 1. Determine target agent from binding rules
    agentName := g.router.Resolve(msg.Channel, msg.Sender, msg.Content)
    
    // 2. Get or activate agent
    agent, err := g.agentManager.Get(ctx, agentName)
    if err != nil {
        return err
    }
    
    // 3. Check permissions
    if !agent.CanUseTool(msg.Intent) {
        return fmt.Errorf("agent %s cannot perform %s", agentName, msg.Intent)
    }
    
    // 4. Forward to agent
    return g.agentManager.Dispatch(ctx, agent, msg)
}
```

**Routing logic:**
- Telegram bot messages → bound agent
- Discord mentions → bound agent
- Cron triggers → scheduled agent
- Manual API calls → specified agent

### 5.2 Agent Manager

```go
package agents

type Manager struct {
    store       AgentStore
    supervisor  Supervisor
    registry    *Registry
}

func (m *Manager) Start(ctx context.Context, agentID string) error {
    agent, err := m.store.Get(ctx, agentID)
    if err != nil {
        return err
    }
    
    // 1. Ensure workspace exists
    if err := m.ensureWorkspace(agent); err != nil {
        return err
    }
    
    // 2. Write config files
    if err := m.writeConfig(agent); err != nil {
        return err
    }
    
    // 3. Start process (Docker / systemd / binary)
    proc, err := m.supervisor.Start(agent)
    if err != nil {
        return err
    }
    
    // 4. Update state
    agent.Status = "active"
    agent.PID = proc.PID
    return m.store.Update(ctx, agent)
}

func (m *Manager) Stop(ctx context.Context, agentID string) error {
    // Graceful shutdown with timeout
}

func (m *Manager) Restart(ctx context.Context, agentID string) error {
    // Stop + Start
}

func (m *Manager) HealthCheck(ctx context.Context, agentID string) (HealthStatus, error) {
    // Ping agent heartbeat endpoint
    // Check process alive
    // Verify last run within threshold
}
```

### 5.3 Supervisor (Process Management)

```go
package supervisor

// Supports multiple backends
type Backend interface {
    Start(agent *Agent) (*Process, error)
    Stop(pid int) error
    Status(pid int) (ProcessStatus, error)
    Logs(pid int, lines int) ([]string, error)
}

// Implementations:
// - DockerBackend (docker run / compose)
// - SystemdBackend (systemctl start/stop)
// - KubernetesBackend (Deployment/StatefulSet)
// - NomadBackend (job dispatch)
// - LocalBackend (os/exec)
```

**Docker Compose example:**
```yaml
# docker-compose.yml for agent fleet
version: '3.8'

services:
  gateway:
    image: openclaw/gateway:latest
    ports:
      - "8080:8080"
    environment:
      DB_HOST: postgres
      RABBITMQ_URL: amqp://rabbitmq:5672
      VAULT_ADDR: http://vault:8200

  ops-agent:
    image: openclaw/agent:latest
    volumes:
      - ./agents/ops:/workspace
    environment:
      AGENT_NAME: ops-agent
      GATEWAY_URL: http://gateway:8080
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  mail-agent:
    image: openclaw/agent:latest
    volumes:
      - ./agents/mail:/workspace
    environment:
      AGENT_NAME: mail-agent
      ALLOWED_TOOLS: "web_search,message,read"
      DENIED_TOOLS: "exec,gateway"
    restart: unless-stopped
```

### 5.4 Approval Service

```go
package approvals

type Service struct {
    store ApprovalStore
    notifier Notifier
}

func (s *Service) RequestApproval(ctx context.Context, req ApprovalRequest) (*Approval, error) {
    // 1. Create approval record
    approval := &Approval{
        RunID: req.RunID,
        AgentID: req.AgentID,
        ActionType: req.ActionType,
        ActionDetails: req.Details,
        Status: "pending",
        ExpiresAt: time.Now().Add(1 * time.Hour),
    }
    
    if err := s.store.Create(ctx, approval); err != nil {
        return nil, err
    }
    
    // 2. Notify human reviewers
    s.notifier.Send(ctx, Notification{
        Channel: "slack",
        Target: "#ops-approvals",
        Message: fmt.Sprintf("Agent %s requests approval to %s: %s", 
            req.AgentName, req.ActionType, req.Details),
        Actions: []Action{
            {Label: "Approve", URL: fmt.Sprintf("/approvals/%s/approve", approval.ID)},
            {Label: "Reject", URL: fmt.Sprintf("/approvals/%s/reject", approval.ID)},
        },
    })
    
    return approval, nil
}

func (s *Service) Approve(ctx context.Context, approvalID string, approvedBy string) error {
    approval, err := s.store.Get(ctx, approvalID)
    if err != nil {
        return err
    }
    
    approval.Status = "approved"
    approval.ApprovedBy = approvedBy
    approval.ApprovedAt = time.Now()
    
    // Notify agent to proceed
    s.eventBus.Publish(ctx, Event{
        Type: "approval.granted",
        ApprovalID: approvalID,
    })
    
    return s.store.Update(ctx, approval)
}
```

### 5.5 Observability Service

```go
package observability

// Metrics collected per run:
type RunMetrics struct {
    AgentID          string
    Model            string
    DurationMs       int
    PromptTokens     int
    CompletionTokens int
    CostUSD          float64
    ToolsCalled      []ToolCall
    Status           string
    Error            string
}

// Dashboard queries:
// - Active agents
// - Runs per agent (last 24h)
// - Error rate per agent
// - Cost per agent
// - Pending approvals
// - Agents needing restart
```

---

## 6. Agent Lifecycle

```
[Inactive]
   │
   │ POST /agents/:id/start
   ▼
[Starting] ──► [Error] (if start fails)
   │
   ▼
[Active] ◄──────┐
   │             │
   │ Heartbeat   │
   │ timeout     │
   ▼             │
[Unhealthy]     │
   │             │
   │ Auto-restart│
   └─────────────┘
   │
   │ POST /agents/:id/stop
   ▼
[Stopping]
   │
   ▼
[Inactive]
```

---

## 7. Permission Model

```
┌─────────────────────────────────────────────────────────┐
│                   Agent Permission                      │
├─────────────────────────────────────────────────────────┤
│  Tool         │ ops-agent │ mail-agent │ calendar-agent │
├───────────────┼───────────┼────────────┼────────────────┤
│  web_search   │    ✅     │     ✅     │      ✅       │
│  message      │    ✅     │     ✅     │      ❌       │
│  read         │    ✅     │     ✅     │      ✅       │
│  exec         │    ✅     │     ❌     │      ❌       │
│  gateway      │    ✅     │     ❌     │      ❌       │
│  cron         │    ✅     │     ✅     │      ✅       │
│  nodes        │    ✅     │     ❌     │      ❌       │
│  image        │    ✅     │     ✅     │      ❌       │
└───────────────┴───────────┴────────────┴────────────────┘
```

**Permission rules:**
- Default: deny all
- Explicit allow list per agent
- Explicit deny list overrides allow list
- Dangerous tools require approval gate

---

## 8. Configuration as Code

```
config/
├── agents/
│   ├── ops-agent/
│   │   ├── agent.json          # Agent definition
│   │   ├── SOUL.md             # Personality
│   │   ├── TOOLS.md            # Tool permissions
│   │   └── bindings.yaml       # Channel routing
│   ├── mail-agent/
│   │   ├── agent.json
│   │   └── ...
│   └── calendar-agent/
│       └── ...
├── prompts/
│   ├── system-prompts/
│   └── task-prompts/
├── global/
│   ├── openclaw.json           # Gateway config
│   ├── models.yaml             # Model aliases and limits
│   └── policies.yaml           # Global approval policies
└── environments/
    ├── dev/
    ├── staging/
    └── prod/
```

**Git sync workflow:**
```bash
# 1. Engineer edits agent config
vim config/agents/ops-agent/TOOLS.md

# 2. Commit and push
git add . && git commit -m "ops-agent: add nodes tool"
git push origin main

# 3. Platform detects change (webhook / poll)
# 4. Platform validates config
# 5. Platform rolls out to affected agents (rolling restart)
```

---

## 9. API Design

### 9.1 Agents

```
GET    /v1/agents              → List agents
POST   /v1/agents              → Create agent
GET    /v1/agents/:id          → Get agent status
PATCH  /v1/agents/:id          → Update agent config
DELETE /v1/agents/:id          → Delete agent

POST   /v1/agents/:id/start    → Start agent
POST   /v1/agents/:id/stop     → Stop agent
POST   /v1/agents/:id/restart  → Restart agent
GET    /v1/agents/:id/logs     → Get agent logs
GET    /v1/agents/:id/runs     → Get run history
```

### 9.2 Runs & Audit

```
GET    /v1/runs                → List runs (filter by agent, status, date)
GET    /v1/runs/:id            → Get run details
GET    /v1/runs/:id/tools      → Get tool calls for run
```

### 9.3 Approvals

```
GET    /v1/approvals           → List pending approvals
GET    /v1/approvals/:id       → Get approval details
POST   /v1/approvals/:id/approve → Approve action
POST   /v1/approvals/:id/reject  → Reject action
```

### 9.4 Channels

```
GET    /v1/channels            → List channels
POST   /v1/channels            → Create channel binding
PATCH  /v1/channels/:id        → Update binding
DELETE /v1/channels/:id        → Remove binding
```

### 9.5 Dashboard

```
GET    /v1/dashboard           → Overview metrics
GET    /v1/dashboard/agents    → Agent health summary
GET    /v1/dashboard/costs     → Cost breakdown
GET    /v1/dashboard/events    → Recent events stream
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
run.approval_granted
run.approval_rejected

tool.called
tool.approved
tool.denied

config.changed
config.synced
config.failed
```

---

## 11. Docker Compose (Local Dev)

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
    volumes:
      - postgres_data:/var/lib/postgresql/data

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
      DB_USER: openclaw
      DB_PASSWORD: openclaw
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672/
      VAULT_ADDR: http://vault:8200
      VAULT_TOKEN: dev-token
    depends_on:
      - postgres
      - rabbitmq
      - vault

  ops-agent:
    image: openclaw/agent:latest
    environment:
      AGENT_NAME: ops-agent
      GATEWAY_URL: http://platform:8080
      ALLOWED_TOOLS: "web_search,message,read,exec,gateway,cron,nodes,image"
    volumes:
      - ./agents/ops:/workspace
    restart: unless-stopped
    depends_on:
      - platform

  mail-agent:
    image: openclaw/agent:latest
    environment:
      AGENT_NAME: mail-agent
      GATEWAY_URL: http://platform:8080
      ALLOWED_TOOLS: "web_search,message,read,cron"
      DENIED_TOOLS: "exec,gateway"
    volumes:
      - ./agents/mail:/workspace
    restart: unless-stopped
    depends_on:
      - platform

volumes:
  postgres_data:
```

---

## 12. MVP Scope

**Phase 1: Core Platform**
- [ ] PostgreSQL schema + migrations
- [ ] Agent CRUD API
- [ ] Process supervisor (Docker backend)
- [ ] Agent start/stop/restart
- [ ] Basic health checks

**Phase 2: Gateway & Routing**
- [ ] Message routing to agents
- [ ] Channel bindings (Telegram, Discord)
- [ ] Agent registry

**Phase 3: Permissions & Approvals**
- [ ] Tool allow/deny lists
- [ ] Approval gate service
- [ ] Slack/email notifications for approvals

**Phase 4: Observability**
- [ ] Run audit logging
- [ ] Event bus (RabbitMQ)
- [ ] Dashboard with agent status

**Phase 5: GitOps**
- [ ] Config as code
- [ ] Git webhook sync
- [ ] Rolling config updates

**Phase 6: Production**
- [ ] Multi-replica support
- [ ] Kubernetes backend
- [ ] Cost tracking
- [ ] Advanced analytics

---

## 13. Directory Structure

```
openclaw-platform/
├── cmd/
│   ├── server/
│   │   └── main.go              # API server
│   └── worker/
│       └── main.go              # Background worker
├── pkg/
│   ├── api/
│   │   ├── http/                # REST handlers
│   │   └── middleware/          # Auth, logging
│   ├── agents/
│   │   ├── manager.go           # Agent lifecycle
│   │   ├── registry.go          # Agent registry
│   │   └── supervisor/
│   │       ├── interface.go
│   │       ├── docker.go
│   │       ├── systemd.go
│   │       └── kubernetes.go
│   ├── gateway/
│   │   ├── router.go            # Message routing
│   │   └── dispatcher.go        # Agent dispatch
│   ├── permissions/
│   │   ├── checker.go           # Tool permission checks
│   │   └── policies.go          # Default policies
│   ├── approvals/
│   │   ├── service.go
│   │   └── notifier.go
│   ├── observability/
│   │   ├── metrics.go
│   │   ├── audit.go
│   │   └── dashboard.go
│   ├── events/
│   │   ├── interface.go
│   │   └── rabbitmq/
│   ├── store/
│   │   ├── interface.go
│   │   └── postgres/
│   ├── secrets/
│   │   ├── interface.go
│   │   └── vault/
│   └── config/
│       └── sync.go              # Git config sync
├── migrations/
├── configs/
│   └── example/
│       ├── agents/
│       ├── prompts/
│       └── policies.yaml
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── Makefile
└── README.md
```

---

## 14. Open Questions

| # | Question | Status |
|---|----------|--------|
| 1 | Auth model: API keys, OAuth, or SSO? | TBD — start with API keys |
| 2 | Multi-tenancy: org/workspace isolation? | TBD — single-tenant first |
| 3 | Cost model: per-agent, per-run, or flat? | Post-MVP |
| 4 | Custom agent images or shared base? | Shared base + volume mounts |
| 5 | Upgrade strategy: rolling or blue/green? | Rolling restart |

---

## References

- OpenClaw docs: https://docs.openclaw.ai
- OpenClaw GitHub: https://github.com/openclaw/openclaw
- OpenClaw config: `/home/leadinnovation/.openclaw/workspace/`
- OpenClaw Gateway: `gateway` tool actions
- River task engine: https://riverqueue.com/
- HashiCorp Vault: https://www.vaultproject.io/
