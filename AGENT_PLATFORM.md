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

## 12. Cost Management & Model Routing

### 12.1 Philosophy: Test on Subscription, Scale on API

As the first customer, start with **subscription-based access** to validate the platform. Once proven, switch to **API-based billing** for cost control at scale.

### 12.2 Model Tiers

| Tier | Type | Models | Cost | Use Case |
|------|------|--------|------|----------|
| **Free** | Subscription | GPT-3.5, Claude Haiku | $0 | Dev testing, simple agents |
| **Plus** | Subscription | GPT-4o, Claude Sonnet | $20/mo/user | Prototyping, low-volume agents |
| **Pro** | Subscription | GPT-4o, Claude Opus | $200/mo/user | Power users, complex reasoning |
| **API** | Pay-per-token | All models | Usage-based | Production, multi-tenant |
| **Enterprise** | Committed | All + custom | Contract | Scale, SLA guarantees |

### 12.3 Switching Strategy

```
Phase 1 (Now):      Subscription (Plus/Pro)
                        ↓ validate platform works
Phase 2 (3-6 mo):    Hybrid (subscription + API fallback)
                        ↓ measure actual usage
Phase 3 (6-12 mo):   API-only with optimization
                        ↓ scale cost-effectively
Phase 4 (12+ mo):    Enterprise/committed use
```

### 12.4 Cost Optimization Layer

```go
package models

type CostManager struct {
    // Per-agent budget tracking
    agentBudgets map[string]*Budget
    
    // Model routing rules
    router *ModelRouter
    
    // Caching layer
    cache *PromptCache
}

type Budget struct {
    AgentID      string
    MonthlyLimit float64      // USD
    CurrentSpend float64
    AlertThreshold float64    // 80% of limit
    
    // Tier settings
    DefaultTier  ModelTier    // Free, Plus, Pro, API
    FallbackTier ModelTier    // Downgrade when budget exceeded
}

func (cm *CostManager) Route(ctx context.Context, task Task, agentID string) (ModelCall, error) {
    budget := cm.agentBudgets[agentID]
    
    // 1. Check budget
    if budget.CurrentSpend >= budget.MonthlyLimit {
        return cm.fallbackToCheapModel(task)
    }
    
    // 2. Check if approaching limit
    if budget.CurrentSpend >= budget.AlertThreshold*budget.MonthlyLimit {
        cm.notifyBudgetWarning(agentID, budget)
    }
    
    // 3. Select model based on task complexity
    model := cm.router.Select(task)
    
    // 4. Check if subscription tier covers this model
    if !cm.tierCoversModel(budget.DefaultTier, model) {
        // Downgrade to cheaper model or switch to API
        model = cm.findAlternative(model, budget.DefaultTier)
    }
    
    return cm.callModel(ctx, model, task)
}

func (cm *CostManager) SelectModel(task Task) string {
    switch task.Complexity {
    case "simple":
        return "gpt-3.5-turbo"        // ~$0.0015/1K tokens
    case "standard":
        return "gpt-4o-mini"          // ~$0.0006/1K tokens
    case "complex":
        return "claude-3.5-sonnet"    // ~$0.003/1K input
    case "coding":
        return "claude-3.5-sonnet"    // Best for code
    case "reasoning":
        return "gpt-4o"               // ~$0.005/1K input
    case "analysis":
        return "kimi/kimi-code"       // Long context
    default:
        return "gpt-4o-mini"          // Default cheap
    }
}
```

### 12.5 Optimization Strategies

| Strategy | Savings | Implementation |
|----------|---------|----------------|
| **Model routing** | 5-10x | Simple tasks → cheap models |
| **Prompt caching** | 2-5x | Cache system prompts, repeated queries |
| **Response caching** | 2-10x | Cache identical requests |
| **Batching** | 50% | OpenAI batch API (24h delay) |
| **Streaming** | 10-20% | Cancel if user disconnects |
| **Context trimming** | 20-40% | Drop old messages, summarize |
| **Function caching** | 30-50% | Cache tool results |

### 12.6 Billing Abstraction

```go
type BillingBackend interface {
    Name() string                    // "openai_api", "anthropic_api", "azure_openai"
    
    // Authentication
    Authenticate(ctx context.Context, creds Credentials) error
    
    // Cost tracking
    TrackUsage(ctx context.Context, usage TokenUsage) error
    GetCurrentSpend(ctx context.Context) (float64, error)
    
    // Model availability
    AvailableModels(ctx context.Context) ([]Model, error)
    GetCostPerToken(model string) (inputCost, outputCost float64, err error)
    
    // Switching
    CanHandle(model string) bool
    EstimateCost(prompt string, model string) (float64, error)
}

// Backends:
// - OpenAISubscriptionBackend (web UI automation - not recommended for production)
// - OpenAIAPIBackend (pay-per-token)
// - AnthropicAPIBackend (pay-per-token)
// - AzureOpenAIBackend (committed use, cheaper)
// - KimiAPIBackend (alternative provider)
```

### 12.7 Cost Dashboard Metrics

```
Per Agent:
- Monthly spend
- Tokens used (input/output)
- Cost per run
- Model distribution
- Budget remaining

Per Team:
- Aggregated spend
- Top agents by cost
- Cost vs value (tasks completed)

Platform-wide:
- Total monthly burn
- Cost per task type
- Optimization opportunities
- Subscription vs API ratio
```

### 12.8 Subscription → API Migration Path

```yaml
# Initial config (subscription phase)
billing:
  mode: subscription          # subscription | api | hybrid
  default_tier: pro           # free | plus | pro
  
  subscription:
    openai:
      tier: pro               # ChatGPT Plus/Pro
      seats: 5                # Number of users
    anthropic:
      tier: pro               # Claude Pro
      seats: 3

  # API fallback for models not in subscription
  api_fallback:
    enabled: true
    provider: openai
    budget_limit: 100         # $100/mo max API spend
    alert_at: 80              # Alert at 80% of budget

---
# Phase 2 config (hybrid)
billing:
  mode: hybrid
  
  subscription:
    openai:
      tier: plus
      seats: 2                # Reduce seats, move to API
  
  api:
    primary: openai
    fallback: anthropic
    budget_per_agent: 50      # $50/mo per agent

---
# Phase 3 config (API-only)
billing:
  mode: api
  provider: azure_openai      # 20-40% cheaper than direct API
  committed_use: 1000         # $1K/mo commitment
  
  cost_controls:
    max_cost_per_run: 0.50    # $0.50 max per request
    daily_budget_per_agent: 10 # $10/day per agent
    model_routing: enabled
    caching: enabled
    batching: enabled
```

## 13. MVP Scope

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

---

## 14. Maintenance & Security

### 14.1 Container Image Updates (Security Patches)

**Automated Image Builds:**

```yaml
# .github/workflows/agent-image-build.yaml
name: Build Agent Images

on:
  push:
    branches: [main]
    paths: ['agents/**', 'Dockerfile']
  schedule:
    - cron: '0 2 * * 1'  # Weekly security rebuild

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Scan base image for vulnerabilities
      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'openclaw/agent:latest'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      # Rebuild with latest security patches
      - name: Build and push
        run: |
          docker build --pull -t openclaw/agent:${{ github.sha }} .
          docker tag openclaw/agent:${{ github.sha }} openclaw/agent:latest
          docker push openclaw/agent:latest
```

**Rolling Update Strategy:**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Start 1 new pod first
      maxUnavailable: 0    # Never drop below desired replicas
```

**Process:**
1. New image pushed with security patches
2. Kubernetes does rolling update: new pod up → old pod down
3. Agent state persisted in PVC (survives restart)
4. Zero downtime

### 14.2 OpenClaw Software Updates

**Semantic Versioning + Channels:**

```yaml
apiVersion: openclaw.io/v1
kind: Agent
metadata:
  name: ops-agent
spec:
  version: "1.2.3"      # Pin version for stability
  updateChannel: "stable"  # stable, beta, canary
```

**Update Channels:**

| Channel | Update Frequency | Risk | Use Case |
|---------|-----------------|------|----------|
| **stable** | Monthly | Low | Production agents |
| **beta** | Weekly | Medium | Staging/testing |
| **canary** | Daily | High | Ops-agent only |

**Automated Updates via Controller:**

```go
// Controller checks for updates
func (r *AgentReconciler) checkForUpdates(ctx context.Context, agent *openclawv1.Agent) error {
    latestVersion := r.versionChecker.GetLatest(agent.Spec.UpdateChannel)
    
    if latestVersion != agent.Status.CurrentVersion {
        agent.Spec.Image = fmt.Sprintf("openclaw/agent:%s", latestVersion)
        
        r.eventRecorder.Eventf(agent, corev1.EventTypeNormal, "Update",
            "Updating from %s to %s", agent.Status.CurrentVersion, latestVersion)
        
        return r.Update(ctx, agent)
    }
    return nil
}
```

### 14.3 Configuration Updates (No Restart Needed)

```bash
# Update ConfigMap
kubectl edit configmap ops-agent-config -n openclaw

# Signal agent to reload (no restart)
kubectl exec -n openclaw deploy/ops-agent -- curl -X POST localhost:8080/reload
```

**Hot reload vs restart:**
- ConfigMap changes → hot reload (0 downtime)
- Image changes → rolling restart
- CRD changes → controller reconciles

### 14.4 Secret Rotation

**API Key Rotation:**

```yaml
# External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: openai-api-key
  namespace: openclaw
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: SecretStore
    name: vault-backend
  target:
    name: openai-api-key
  data:
    - secretKey: api-key
      remoteRef:
        key: providers/openai
        property: api-key
```

**Rotation workflow:**
1. Update secret in Vault
2. External Secrets Operator detects change
3. Kubernetes Secret updated
4. Agent reads new secret on next API call
5. No restart needed

### 14.5 Maintenance Windows

**Automated Maintenance Schedule:**

```yaml
apiVersion: openclaw.io/v1
kind: MaintenanceWindow
metadata:
  name: weekly-patches
spec:
  schedule: "0 3 * * 0"  # Sunday 3 AM
  duration: 2h
  agents:
    - mail-agent
    - calendar-agent
  exclude:
    - ops-agent  # Never restart ops-agent automatically
  actions:
    - imageUpdate
    - configSync
    - workspaceCleanup
```

**Pre-Maintenance Checklist:**

```bash
# 1. Backup all agent workspaces
kubectl exec -n openclaw deploy/ops-agent -- tar czf /backup/workspace-$(date +%s).tar.gz /workspace/

# 2. Check for pending approvals
curl http://gateway.openclaw.svc.cluster.local:8080/v1/approvals?status=pending

# 3. Drain queue
kubectl scale agent mail-agent --replicas=0  # Stop accepting new work
sleep 60  # Wait for in-flight to complete

# 4. Apply updates
kubectl rollout restart deployment/mail-agent -n openclaw

# 5. Verify health
kubectl rollout status deployment/mail-agent -n openclaw
curl http://mail-agent.openclaw.svc.cluster.local:8080/health

# 6. Restore traffic
kubectl scale agent mail-agent --replicas=1
```

### 14.6 Self-Healing

**Automatic Restart on Failure:**

```yaml
spec:
  template:
    spec:
      containers:
        - name: agent
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 30
            failureThreshold: 3  # Restart after 3 failures
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

**Circuit Breaker for External APIs:**

```go
// If OpenAI API is failing, switch to fallback model
func (a *Agent) CallModel(ctx context.Context, prompt string) (string, error) {
    resp, err := a.primaryModel.Call(ctx, prompt)
    if err != nil {
        if a.failureCount > 3 {
            log.Warn("Primary model failing, switching to fallback")
            return a.fallbackModel.Call(ctx, prompt)
        }
        a.failureCount++
        return "", err
    }
    a.failureCount = 0
    return resp, nil
}
```

### 14.7 Security Hardening

**Distroless Images:**

```dockerfile
# Dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o agent ./cmd/agent

FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/agent /agent
COPY --from=builder /app/config /config
USER 65532:65532  # Non-root
ENTRYPOINT ["/agent"]
```

**Read-Only Root Filesystem:**

```yaml
securityContext:
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  runAsUser: 65532
  capabilities:
    drop:
      - ALL
```

**Network Policies:**

```yaml
# Block all egress except HTTPS and internal services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agent-egress
spec:
  podSelector:
    matchLabels:
      app: agent
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: openclaw
    - to: []
      ports:
        - protocol: TCP
          port: 443
```

### 14.8 Monitoring Maintenance Health

**Prometheus Alerts:**

```yaml
# prometheus-rules.yaml
groups:
  - name: agent-maintenance
    rules:
      - alert: AgentImageOutdated
        expr: openclaw_agent_image_age_hours > 168  # 7 days
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Agent {{ $labels.agent }} image is outdated"

      - alert: AgentUpdateFailed
        expr: increase(openclaw_agent_update_failures[1h]) > 0
        labels:
          severity: critical
        annotations:
          summary: "Agent {{ $labels.agent }} update failed"

      - alert: HighVulnerabilityCount
        expr: openclaw_agent_vulnerabilities > 5
        labels:
          severity: critical
        annotations:
          summary: "Agent image has {{ $value }} vulnerabilities"
```

### 14.9 Maintenance Best Practices

| Task | Method | Downtime |
|------|--------|----------|
| Security patches | Rolling update | Zero |
| Config changes | Hot reload | Zero |
| Secret rotation | Vault + External Secrets | Zero |
| Major version | Canary deployment | Minimal |
| Workspace cleanup | CronJob | Zero |
| Emergency restart | kubectl rollout restart | ~30s |

**Golden rule:** Treat agents like cattle, not pets. Kubernetes handles the rest.
