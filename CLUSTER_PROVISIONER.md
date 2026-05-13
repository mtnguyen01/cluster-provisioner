# Cluster Provisioner Service — Architecture Plan

**Goal:** A production Go service that provisions Kubernetes clusters on multiple hosting providers.
**Backbone:** PostgreSQL + RabbitMQ
**Date:** 2026-05-13

---

## 1. Why Not Cluster API?

[Cluster API (CAPI)](https://cluster-api.sigs.k8s.io/) already exists. We evaluated it and decided against it because:

1. **CAPI runs inside Kubernetes.** We want a standalone service.
2. **Talos-specific needs.** Rescue mode, raw image flashing, provider quirks don't fit CAPI's declarative model.
3. **Addon orchestration.** Helm-based addon installation with dependency ordering.
4. **Multi-provider abstraction.** Single API for Hetzner, Scaleway, Vultr.

**Trade-off:** We hand-roll the reconciler. Acceptable for ~3 providers and ~3 cluster types.

---

## 2. Production Backbone

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Cluster Provisioner Service                      │
│                     (Go, multi-replica capable)                     │
└─────────────────────────────────────────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
       ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
       │  PostgreSQL  │ │   RabbitMQ   │ │    Vault     │
       │   (state)    │ │   (events)   │ │  (secrets)   │
       └──────────────┘ └──────────────┘ └──────────────┘
```

| Component | Role |
|-----------|------|
| **PostgreSQL** | Primary state store |
| **RabbitMQ** | Event bus + work queue |
| **HashiCorp Vault** | Secret management |

---

## 3. Architecture Layers

```
┌─────────────────────────────────────────────────────────┐
│              API Layer (gRPC + REST)                    │
├─────────────────────────────────────────────────────────┤
│           Reconciler (declarative)                      │
├─────────────────────────────────────────────────────────┤
│           Task Engine (River / durable)                 │
├─────────────────────────────────────────────────────────┤
│           Capability-Based Provider Interface           │
├─────────────────────────────────────────────────────────┤
│           Cluster Engine Interface                      │
├─────────────────────────────────────────────────────────┤
│           Addon Manager (with dependencies)             │
├─────────────────────────────────────────────────────────┤
│           Cloud SDKs (Go libraries)                     │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Data Model

### clusters

```sql
CREATE TABLE clusters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE,
    provider TEXT NOT NULL,
    region TEXT NOT NULL,
    cluster_type TEXT NOT NULL,
    control_planes INT NOT NULL DEFAULT 1,
    workers INT NOT NULL DEFAULT 1,
    vpc_cidr TEXT,
    use_private_ip BOOLEAN DEFAULT true,
    vpn_type TEXT,
    phase TEXT NOT NULL DEFAULT 'Pending',
    current_step TEXT,
    conditions JSONB DEFAULT '{}',
    failure_reason TEXT,
    retry_count INT DEFAULT 0,
    spec JSONB NOT NULL,
    state JSONB DEFAULT '{}',
    public_endpoint TEXT,
    private_endpoint TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ,
    deleted_at TIMESTAMPTZ
);
```

### nodes

```sql
CREATE TABLE nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id UUID NOT NULL REFERENCES clusters(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    role TEXT NOT NULL,
    public_ip INET,
    private_ip INET,
    vpn_ip INET,
    provider_id TEXT,
    status TEXT NOT NULL DEFAULT 'Creating',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);
```

### resources

```sql
CREATE TABLE resources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id UUID NOT NULL REFERENCES clusters(id) ON DELETE CASCADE,
    resource_type TEXT NOT NULL,
    provider TEXT NOT NULL,
    provider_id TEXT NOT NULL,
    name TEXT NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);
```

---

## 5. Provider Interface (Capability-Based)

```go
package providers

type Provider interface {
    Name() string
    Capabilities() Capabilities
}

type Capabilities struct {
    HasVMLifecycle      bool
    HasRescueMode       bool
    HasPrivateNetwork   bool
    HasFirewall         bool
    HasLoadBalancer     bool
    HasVolumes          bool
}

type VMLifecycler interface {
    CreateVM(ctx context.Context, opts VMOptions) (*VM, error)
    DeleteVM(ctx context.Context, id string) error
    GetVM(ctx context.Context, id string) (*VM, error)
    ListVMs(ctx context.Context, filters VMFilters) ([]VM, error)
    WaitForVM(ctx context.Context, id string, desired VMStatus, timeout time.Duration) error
}

type Networker interface {
    CreateNetwork(ctx context.Context, opts NetworkOptions) (*Network, error)
    DeleteNetwork(ctx context.Context, id string) error
    AttachNetwork(ctx context.Context, vmID, networkID string) error
}

type RescueBootable interface {
    EnableRescueMode(ctx context.Context, vmID string) error
    DisableRescueMode(ctx context.Context, vmID string) error
}
```

---

## 6. Cluster Engine (Reconciler-Based)

```go
package clusters

type Engine interface {
    Name() string
    ValidateSpec(spec ClusterSpec) error
    Plan(ctx context.Context, cluster *Cluster, provider providers.Provider) ([]Step, error)
    HealthCheck(ctx context.Context, cluster *Cluster) (HealthStatus, error)
    Destroy(ctx context.Context, cluster *Cluster, provider providers.Provider) error
}

type Step struct {
    ID         string
    Name       string
    Type       StepType
    DependsOn  []string
    Execute    func(ctx context.Context) error
    Compensate func(ctx context.Context) error
    Timeout    time.Duration
    MaxRetries int
}
```

### Talos Engine Plan

1. Create network
2. Create control plane VMs
3. Install Talos (rescue mode → flash image)
4. Generate and apply Talos config
5. Bootstrap etcd
6. Wait for K8s API
7. Join workers
8. Install CNI

---

## 7. Addon Interface

```go
package addons

type Installer interface {
    Name() string
    Dependencies() []string
    DefaultConfig(provider string, clusterType string) map[string]string
    ValidateConfig(config map[string]string) error
    IsInstalled(ctx context.Context, cluster *Cluster) (bool, error)
    Install(ctx context.Context, cluster *Cluster, config map[string]string) error
    Upgrade(ctx context.Context, cluster *Cluster, config map[string]string) error
    Uninstall(ctx context.Context, cluster *Cluster) error
    Status(ctx context.Context, cluster *Cluster) (AddonStatus, error)
}
```

---

## 8. API

```
POST   /v1/clusters              → Create cluster (202 Accepted)
GET    /v1/clusters              → List clusters
GET    /v1/clusters/:id          → Get cluster status
DELETE /v1/clusters/:id          → Delete cluster
GET    /v1/clusters/:id/events   → SSE stream
GET    /v1/clusters/:id/kubeconfig → Download kubeconfig
```

---

## 9. Configuration

```yaml
database:
  type: postgres
  host: ${DB_HOST}
  port: 5432
  database: cluster_provisioner

messaging:
  type: rabbitmq
  url: ${RABBITMQ_URL}

secrets:
  type: vault
  address: ${VAULT_ADDR}

task_engine:
  type: river
  max_workers: 10

providers:
  hetzner:
    token_path: providers/hetzner/token
  scaleway:
    credentials_path: providers/scaleway/credentials
```

---

## 10. Cost Management

### 10.1 Subscription → API Strategy

For early testing, use subscription tiers (ChatGPT Plus/Claude Pro). For production, switch to API with cost optimization.

```yaml
# Phase 1: Subscription (testing)
billing:
  mode: subscription
  tiers:
    openai: plus      # $20/mo
    anthropic: pro    # $20/mo

# Phase 2: Hybrid (transition)
billing:
  mode: hybrid
  subscription:
    openai: plus      # Keep for human testing
  api:
    provider: openai  # For automated provisioning
    budget_limit: 200  # $200/mo

# Phase 3: API-only (production)
billing:
  mode: api
  provider: azure_openai  # 20-40% cheaper
  committed_use: 1000     # $1K/mo commitment
  
  cost_controls:
    max_cost_per_provision: 5.00    # $5 max per cluster
    daily_budget: 50                 # $50/day platform-wide
```

### 10.2 Cost Optimization

| Strategy | Savings | Implementation |
|----------|---------|----------------|
| Model routing | 5-10x | Simple tasks → GPT-3.5 |
| Prompt caching | 2-5x | Cache repeated queries |
| Response caching | 2-10x | Cache identical requests |
| Batching | 50% | Batch API for non-urgent |
| Context trimming | 20-40% | Drop old messages |

## 11. MVP Scope

- [ ] PostgreSQL schema + migrations
- [ ] Hetzner + Scaleway providers
- [ ] Talos engine with step plan
- [ ] Cilium + MetalLB addons
- [ ] River task engine
- [ ] REST API
- [ ] Vault secret management
