# Cluster Provisioner Service — Architecture Plan v2

**Goal:** A production Go service that provisions Kubernetes clusters on multiple hosting providers.
**Backbone:** PostgreSQL + RabbitMQ
**Inspiration:** `poc-hetzner` (Talos on Hetzner/Scaleway/Vultr with VPN, CNI, ingress)

**Date:** 2026-05-13

---

## 1. Why Not Cluster API?

[Cluster API (CAPI)](https://cluster-api.sigs.k8s.io/) already exists with providers for Hetzner ([CAPH](https://github.com/syself/cluster-api-provider-hetzner)) and others. We evaluated it and decided against it because:

1. **CAPI runs inside Kubernetes.** We want a standalone service that doesn't require a management cluster.
2. **Talos-specific needs.** Rescue mode, raw image flashing, and provider-specific quirks (Scaleway's IAM SSH keys, Vultr's VPC attachment) don't fit CAPI's declarative model cleanly.
3. **Addon orchestration.** We need Helm-based addon installation (Cilium, Envoy Gateway, cert-manager) with dependency ordering and config templating — outside CAPI's scope.
4. **Multi-provider abstraction.** We want a single API that abstracts Hetzner, Scaleway, Vultr, and future providers with unified networking/VPN patterns.

**Trade-off:** We hand-roll the reconciler and state model. This is acceptable for a focused service with ~3 providers and ~3 cluster types.

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
       │   PostgreSQL │ │   RabbitMQ   │ │   Vault      │
       │   (state)    │ │   (events)   │ │  (secrets)   │
       └──────────────┘ └──────────────┘ └──────────────┘
```

| Component | Role | Why |
|-----------|------|-----|
| **PostgreSQL** | Primary state store | ACID transactions, querying, migrations, multi-replica safe |
| **RabbitMQ** | Event bus + work queue | Durable task queues, pub/sub for events, dead-letter for retries |
| **HashiCorp Vault** | Secret management | Kubeconfig, cloud API tokens, SSH keys — never in DB plaintext |

---

## 3. Architecture Layers

```
┌─────────────────────────────────────────────────────────┐
│              API Layer (gRPC + REST)                    │
│  CreateCluster, GetCluster, DeleteCluster, StreamEvents │
├─────────────────────────────────────────────────────────┤
│           Reconciler (declarative, not imperative)      │
│  Diff desired vs observed state, emit tasks, converge   │
├─────────────────────────────────────────────────────────┤
│           Task Engine (River / Asynq over RabbitMQ)     │
│  Durable execution: tasks survive process restarts      │
├─────────────────────────────────────────────────────────┤
│           Capability-Based Provider Interface           │
│  VM lifecycler, Networker, RescueBootable, etc.       │
├─────────────────────────────────────────────────────────┤
│           Cluster Engine Interface                      │
│  Talos, k3s, RKE2 — each with its own reconcile plan   │
├─────────────────────────────────────────────────────────┤
│           Addon Manager (with dependencies)             │
│  Ordered install: CNI → CCM → Ingress → CertManager   │
├─────────────────────────────────────────────────────────┤
│           Cloud SDKs (Go libraries, not shell)          │
│  Hetzner SDK, Scaleway SDK, Vultr SDK                   │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Data Model (PostgreSQL)

### 4.1 clusters

```sql
CREATE TABLE clusters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE,
    provider TEXT NOT NULL,       -- 'hetzner', 'scaleway', 'vultr'
    region TEXT NOT NULL,
    cluster_type TEXT NOT NULL,   -- 'talos', 'k3s', 'rke2'
    
    control_planes INT NOT NULL DEFAULT 1,
    workers INT NOT NULL DEFAULT 1,
    
    vpc_cidr TEXT,
    use_private_ip BOOLEAN DEFAULT true,
    vpn_type TEXT,                -- 'netbird', 'tailscale', 'headscale', null
    
    -- State
    phase TEXT NOT NULL DEFAULT 'Pending',
    current_step TEXT,            -- exact resumable step
    conditions JSONB DEFAULT '{}', -- { "NetworkReady": true, "NodesReady": false }
    failure_reason TEXT,
    retry_count INT DEFAULT 0,
    last_error TEXT,
    
    -- Endpoints (not secrets)
    public_endpoint TEXT,
    private_endpoint TEXT,
    
    -- Metadata
    spec JSONB NOT NULL,          -- full user spec
    state JSONB DEFAULT '{}',     -- provider-specific runtime state
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ,
    deleted_at TIMESTAMPTZ        -- soft delete
);

CREATE INDEX idx_clusters_phase ON clusters(phase);
CREATE INDEX idx_clusters_provider ON clusters(provider);
CREATE INDEX idx_clusters_expires_at ON clusters(expires_at) WHERE expires_at IS NOT NULL;
```

### 4.2 nodes

```sql
CREATE TABLE nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id UUID NOT NULL REFERENCES clusters(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    role TEXT NOT NULL,           -- 'control-plane', 'worker', 'vpn'
    
    -- IPs
    public_ip INET,
    private_ip INET,
    vpn_ip INET,
    
    -- Provider
    provider_id TEXT,             -- cloud VM ID
    provider_state JSONB,         -- provider-specific VM metadata
    
    -- State
    status TEXT NOT NULL DEFAULT 'Creating',
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,
    
    UNIQUE(cluster_id, name)
);

CREATE INDEX idx_nodes_cluster_id ON nodes(cluster_id);
```

### 4.3 resources (VMs, networks, volumes, load balancers)

```sql
CREATE TABLE resources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id UUID NOT NULL REFERENCES clusters(id) ON DELETE CASCADE,
    node_id UUID REFERENCES nodes(id) ON DELETE SET NULL,
    
    resource_type TEXT NOT NULL,  -- 'vm', 'network', 'volume', 'firewall', 'loadbalancer', 'ssh_key'
    provider TEXT NOT NULL,
    provider_id TEXT NOT NULL,    -- cloud resource ID
    
    -- For cleanup tracking
    name TEXT NOT NULL,
    metadata JSONB DEFAULT '{}',
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,
    
    UNIQUE(provider, provider_id)
);

CREATE INDEX idx_resources_cluster_id ON resources(cluster_id);
CREATE INDEX idx_resources_type ON resources(resource_type);
```

### 4.4 addons

```sql
CREATE TABLE addons (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id UUID NOT NULL REFERENCES clusters(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    version TEXT,
    config JSONB DEFAULT '{}',
    
    status TEXT NOT NULL DEFAULT 'Pending',
    installed_at TIMESTAMPTZ,
    
    UNIQUE(cluster_id, name)
);

CREATE INDEX idx_addons_cluster_id ON addons(cluster_id);
```

### 4.5 tasks (for River/Asynq integration)

```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id UUID NOT NULL REFERENCES clusters(id) ON DELETE CASCADE,
    
    task_type TEXT NOT NULL,      -- 'create_vm', 'install_cni', 'bootstrap_etcd'
    payload JSONB NOT NULL,
    
    status TEXT NOT NULL DEFAULT 'Pending',
    attempts INT DEFAULT 0,
    max_attempts INT DEFAULT 3,
    
    scheduled_at TIMESTAMPTZ DEFAULT NOW(),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    error_message TEXT,
    
    -- For ordering within a cluster
    depends_on UUID[],
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_tasks_cluster_id ON tasks(cluster_id);
CREATE INDEX idx_tasks_status_scheduled ON tasks(status, scheduled_at);
```

---

## 5. Provider Interface (Capability-Based)

Instead of a monolithic interface, providers implement **capability interfaces**.

```go
package providers

// Base — every provider must implement
type Provider interface {
    Name() string
    Capabilities() Capabilities
}

type Capabilities struct {
    HasVMLifecycle      bool
    HasRescueMode       bool
    HasCustomImageBoot  bool
    HasPrivateNetwork   bool
    HasFirewall         bool
    HasLoadBalancer     bool
    HasVolumes          bool
    HasCCM              bool
    HasCSI              bool
}

// VM Lifecycle — required for all providers
type VMLifecycler interface {
    CreateVM(ctx context.Context, opts VMOptions) (*VM, error)
    DeleteVM(ctx context.Context, id string) error
    GetVM(ctx context.Context, id string) (*VM, error)
    ListVMs(ctx context.Context, filters VMFilters) ([]VM, error)
    WaitForVM(ctx context.Context, id string, desired VMStatus, timeout time.Duration) error
}

// Network — for providers with private networking
type Networker interface {
    CreateNetwork(ctx context.Context, opts NetworkOptions) (*Network, error)
    DeleteNetwork(ctx context.Context, id string) error
    AttachNetwork(ctx context.Context, vmID, networkID string) error
    DetachNetwork(ctx context.Context, vmID, networkID string) error
}

// Rescue Mode — for providers that support rescue boot (Hetzner, Scaleway)
type RescueBootable interface {
    EnableRescueMode(ctx context.Context, vmID string) error
    DisableRescueMode(ctx context.Context, vmID string) error
}

// Firewall
type FirewallManager interface {
    CreateFirewall(ctx context.Context, opts FirewallOptions) (*Firewall, error)
    DeleteFirewall(ctx context.Context, id string) error
    AttachFirewall(ctx context.Context, fwID, vmID string) error
}

// Load Balancer
type LoadBalancerManager interface {
    CreateLoadBalancer(ctx context.Context, opts LBOptions) (*LoadBalancer, error)
    DeleteLoadBalancer(ctx context.Context, id string) error
}

// SSH Keys
type SSHKeyManager interface {
    UploadSSHKey(ctx context.Context, name, publicKey string) (*SSHKey, error)
    DeleteSSHKey(ctx context.Context, id string) error
    ListSSHKeys(ctx context.Context) ([]SSHKey, error)
}

// Volumes
type VolumeManager interface {
    CreateVolume(ctx context.Context, opts VolumeOptions) (*Volume, error)
    AttachVolume(ctx context.Context, vmID, volumeID string) error
    DeleteVolume(ctx context.Context, id string) error
}
```

### Provider Registry

```go
type Registry struct {
    providers map[string]Provider
}

func (r *Registry) Register(name string, p Provider) { ... }
func (r *Registry) Get(name string) (Provider, error) { ... }

// Type assertions for capabilities
func HasCapability[T any](p Provider) (T, bool) {
    cap, ok := p.(T)
    return cap, ok
}

// Usage:
if rescuer, ok := providers.HasCapability[RescueBootable](provider); ok {
    rescuer.EnableRescueMode(ctx, vmID)
}
```

---

## 6. Cluster Engine Interface (Reconciler-Based)

Instead of monolithic `Bootstrap()`, engines produce a **reconcile plan** of discrete steps.

```go
package clusters

// Engine generates a reconciliation plan for a cluster type
type Engine interface {
    Name() string
    Description() string
    
    // Validation
    ValidateSpec(spec ClusterSpec) error
    
    // Generate a reconcile plan from current state to desired state
    Plan(ctx context.Context, cluster *Cluster, provider providers.Provider) ([]Step, error)
    
    // Health check
    HealthCheck(ctx context.Context, cluster *Cluster) (HealthStatus, error)
    
    // Cleanup
    Destroy(ctx context.Context, cluster *Cluster, provider providers.Provider) error
}

// Step is a discrete, retryable, compensatable unit of work
type Step struct {
    ID            string
    Name          string
    Type          StepType     // CreateVM, InstallOS, BootstrapEtcd, InstallAddon
    DependsOn     []string     // step IDs that must complete first
    
    // Execution
    Execute       func(ctx context.Context) error
    
    // Compensation (rollback on failure)
    Compensate    func(ctx context.Context) error
    
    // Config
    Timeout       time.Duration
    MaxRetries    int
    
    // Result
    Result        StepResult
}

type StepType string
const (
    StepCreateNetwork   StepType = "create_network"
    StepCreateVM        StepType = "create_vm"
    StepInstallOS       StepType = "install_os"
    StepGenerateConfig  StepType = "generate_config"
    StepBootstrapEtcd   StepType = "bootstrap_etcd"
    StepJoinWorkers     StepType = "join_workers"
    StepInstallAddon    StepType = "install_addon"
    StepConfigureVPN    StepType = "configure_vpn"
    StepValidate        StepType = "validate"
)

type StepResult struct {
    Status    StepStatus  // Pending, Running, Completed, Failed, Compensated
    Output    map[string]interface{}
    Error     string
    StartedAt *time.Time
    EndedAt   *time.Time
}
```

### Talos Engine Plan Example

```go
func (e *TalosEngine) Plan(ctx context.Context, cluster *Cluster, p providers.Provider) ([]Step, error) {
    steps := []Step{}
    
    // Step 1: Ensure network exists
    steps = append(steps, Step{
        ID: "network",
        Name: "Create VPC Network",
        Type: StepCreateNetwork,
        Execute: func(ctx context.Context) error { ... },
        Compensate: func(ctx context.Context) error { 
            return networker.DeleteNetwork(ctx, networkID) 
        },
    })
    
    // Step 2: Create control plane VMs
    for i := 0; i < cluster.ControlPlanes; i++ {
        steps = append(steps, Step{
            ID: fmt.Sprintf("cp-%d", i),
            Name: fmt.Sprintf("Create CP%d", i+1),
            Type: StepCreateVM,
            DependsOn: []string{"network"},
            Execute: func(ctx context.Context) error { ... },
            Compensate: func(ctx context.Context) error { 
                return vmlc.DeleteVM(ctx, vmID) 
            },
        })
    }
    
    // Step 3: Install OS (rescue mode for Talos)
    for i := 0; i < cluster.ControlPlanes; i++ {
        steps = append(steps, Step{
            ID: fmt.Sprintf("cp-%d-os", i),
            Name: fmt.Sprintf("Install Talos on CP%d", i+1),
            Type: StepInstallOS,
            DependsOn: []string{fmt.Sprintf("cp-%d", i)},
            Execute: func(ctx context.Context) error { 
                // Enable rescue, flash image, reboot
                return e.installTalos(ctx, vmID, providerID) 
            },
            Compensate: func(ctx context.Context) error { 
                // Delete and recreate VM
                return nil 
            },
        })
    }
    
    // Step 4: Generate and apply Talos config
    steps = append(steps, Step{
        ID: "talos-config",
        Name: "Generate and Apply Talos Config",
        Type: StepGenerateConfig,
        DependsOn: []string{"cp-0-os", "cp-1-os"}, // all CPs installed
        Execute: func(ctx context.Context) error { ... },
    })
    
    // Step 5: Bootstrap etcd
    steps = append(steps, Step{
        ID: "etcd",
        Name: "Bootstrap etcd",
        Type: StepBootstrapEtcd,
        DependsOn: []string{"talos-config"},
        Execute: func(ctx context.Context) error { ... },
    })
    
    // Step 6: Wait for K8s API
    steps = append(steps, Step{
        ID: "k8s-api",
        Name: "Wait for Kubernetes API",
        Type: StepValidate,
        DependsOn: []string{"etcd"},
        Execute: func(ctx context.Context) error { ... },
    })
    
    // Step 7: Join workers
    for i := 0; i < cluster.Workers; i++ {
        steps = append(steps, Step{
            ID: fmt.Sprintf("worker-%d", i),
            Name: fmt.Sprintf("Create and Join Worker %d", i+1),
            Type: StepJoinWorkers,
            DependsOn: []string{"k8s-api"},
            Execute: func(ctx context.Context) error { ... },
        })
    }
    
    // Step 8: Install CNI
    steps = append(steps, Step{
        ID: "cni",
        Name: "Install CNI",
        Type: StepInstallAddon,
        DependsOn: []string{"k8s-api"},
        Execute: func(ctx context.Context) error { 
            return addonManager.Install(ctx, cluster, "cilium", config) 
        },
    })
    
    return steps, nil
}
```

---

## 7. Addon Interface (With Dependencies)

```go
package addons

type Installer interface {
    Name() string
    Description() string
    
    // Dependencies — addons that must be installed first
    Dependencies() []string
    
    // Default config for this addon given provider + cluster
    DefaultConfig(provider string, clusterType string) map[string]string
    
    // Validation
    ValidateConfig(config map[string]string) error
    
    // Lifecycle
    IsInstalled(ctx context.Context, cluster *Cluster) (bool, error)
    Install(ctx context.Context, cluster *Cluster, config map[string]string) error
    Upgrade(ctx context.Context, cluster *Cluster, config map[string]string) error
    Uninstall(ctx context.Context, cluster *Cluster) error
    
    // Status
    Status(ctx context.Context, cluster *Cluster) (AddonStatus, error)
    
    // Required secrets (Vault paths)
    RequiredSecrets() []string
}

type AddonStatus struct {
    Phase      AddonPhase  // Pending, Installing, Ready, Failed
    Version    string
    Message    string
    Conditions map[string]bool
}

// Example: Cilium depends on nothing
// Example: MetalLB depends on CNI
// Example: Cloudflare Tunnel depends on Ingress + CertManager
// Example: CertManager has no dependencies but is depended on by others
```

---

## 8. Task Engine (River over RabbitMQ)

Instead of an in-process goroutine pool, use **durable task execution** that survives restarts.

### Options Evaluated

| Engine | Pros | Cons |
|--------|------|------|
| **Temporal** | Full saga, visibility UI, mature | Heavy, requires separate server |
| **River** | Postgres-native, Go-native, lightweight | Younger project |
| **Asynq** | Redis-based, simple | Redis not our backbone |
| **Custom RabbitMQ** | Fits our backbone, flexible | More code to maintain |

**Decision:** Use **River** (or **Asynq** if we add Redis later) for the task engine, with RabbitMQ as the event bus for pub/sub.

### Task Types

```go
package tasks

// Provisioning tasks

const (
    TypeCreateCluster    = "cluster:create"
    TypeDeleteCluster    = "cluster:delete"
    TypeInstallAddon     = "addon:install"
    TypeExecuteStep      = "step:execute"
    TypeCompensateStep   = "step:compensate"
    TypeHealthCheck      = "cluster:health_check"
    TypeAutoTeardown     = "cluster:auto_teardown"
)

// River job handler
func (w *Worker) Work(ctx context.Context, job *river.Job[CreateClusterArgs]) error {
    cluster, err := w.store.GetCluster(ctx, job.Args.ClusterID)
    if err != nil {
        return err
    }
    
    // Generate plan
    engine := w.engines.Get(cluster.ClusterType)
    steps, err := engine.Plan(ctx, cluster, w.providers.Get(cluster.Provider))
    
    // Execute each step
    for _, step := range steps {
        if err := w.executeStep(ctx, cluster, step); err != nil {
            // Trigger compensation
            return w.compensate(ctx, cluster, steps, step)
        }
    }
    
    return nil
}
```

### Event Bus (RabbitMQ)

```go
package events

type Bus interface {
    Publish(ctx context.Context, event Event) error
    Subscribe(queue string, handler EventHandler) error
}

type Event struct {
    Type      string      // "cluster.created", "step.completed", "step.failed"
    ClusterID string
    Timestamp time.Time
    Payload   map[string]interface{}
}

// Event types:
// - cluster.provisioning.started
// - cluster.step.started
// - cluster.step.completed
// - cluster.step.failed
// - cluster.step.compensated
// - cluster.ready
// - cluster.failed
// - cluster.deleting
// - cluster.deleted
// - addon.installing
// - addon.ready
```

---

## 9. API Design

### 9.1 REST API (v1)

```
POST   /v1/clusters              → Create cluster (202 Accepted, returns operation ID)
GET    /v1/clusters              → List clusters
GET    /v1/clusters/:id          → Get cluster status
PATCH  /v1/clusters/:id          → Update cluster (scale, addons)
DELETE /v1/clusters/:id          → Delete cluster (202 Accepted)
POST   /v1/clusters/:id/cancel   → Cancel ongoing operation
POST   /v1/clusters/:id/retry    → Retry failed operation

GET    /v1/clusters/:id/events      → SSE stream of provisioning events
GET    /v1/clusters/:id/nodes       → List nodes
GET    /v1/clusters/:id/resources   → List cloud resources
GET    /v1/clusters/:id/addons      → List addons
POST   /v1/clusters/:id/addons      → Install addon
DELETE /v1/clusters/:id/addons/:name → Uninstall addon

GET    /v1/clusters/:id/kubeconfig  → Download kubeconfig ( Vault-secured )

GET    /v1/providers             → List providers
GET    /v1/providers/:name/capabilities → Provider capabilities
GET    /v1/cluster-types         → List cluster types
GET    /v1/addons                → List available addons
```

### 9.2 Idempotency

All mutating endpoints accept an `Idempotency-Key: <uuid>` header. The service stores `(key, response, expiration)` in Redis for 24h.

```go
func (s *Server) CreateCluster(w http.ResponseWriter, r *http.Request) {
    key := r.Header.Get("Idempotency-Key")
    if key != "" {
        if resp, ok := s.idempotencyStore.Get(key); ok {
            writeJSON(w, resp.StatusCode, resp.Body)
            return
        }
    }
    
    // Process request
    cluster, err := s.service.CreateCluster(r.Context(), req)
    
    if key != "" {
        s.idempotencyStore.Set(key, resp, 24*time.Hour)
    }
}
```

---

## 10. Secret Management (HashiCorp Vault)

**Nothing sensitive in PostgreSQL.**

| Secret | Vault Path | Access |
|--------|-----------|--------|
| Cloud API tokens | `providers/hetzner/token` | Service account |
| Cloud API tokens | `providers/scaleway/credentials` | Service account |
| SSH private keys | `clusters/:id/ssh-keys/:name` | Service account |
| Kubeconfig | `clusters/:id/kubeconfig` | Service account + cluster owner |
| Netbird setup key | `vpn/netbird/setup-key` | Service account |
| Cloudflare token | `dns/cloudflare/token` | Service account |
| Let's Encrypt | `certs/letsencrypt/account` | Service account |

```go
type SecretStore interface {
    Get(ctx context.Context, path string) (string, error)
    Put(ctx context.Context, path string, value string) error
    Delete(ctx context.Context, path string) error
    GetKubeconfig(ctx context.Context, clusterID string) ([]byte, error)
}
```

---

## 11. Configuration

```yaml
# config.yaml
server:
  http_port: 8080
  grpc_port: 9090
  idempotency_ttl: 24h

database:
  type: postgres
  host: ${DB_HOST}
  port: 5432
  database: cluster_provisioner
  user: ${DB_USER}
  password: ${DB_PASSWORD}
  ssl_mode: require
  pool_size: 20

messaging:
  type: rabbitmq
  url: ${RABBITMQ_URL}
  exchange: cluster-provisioner.events
  queues:
    - cluster-events
    - task-retries
    - dead-letter

secrets:
  type: vault
  address: ${VAULT_ADDR}
  token: ${VAULT_TOKEN}
  mount_path: secret

providers:
  hetzner:
    token_path: providers/hetzner/token
  scaleway:
    credentials_path: providers/scaleway/credentials
  vultr:
    api_key_path: providers/vultr/api-key

task_engine:
  type: river
  max_workers: 10
  poll_interval: 5s
  
  queues:
    - name: provisioning
      priority: 1
      max_concurrent: 5
    - name: health_checks
      priority: 2
      max_concurrent: 10

addons:
  cilium:
    default_version: v1.19.3
  metallb:
    enabled: true
  envoy_gateway:
    default_version: v1.1.0
  cert_manager:
    default_version: v1.14.0

provisioning:
  default_ttl: 24h
  max_ttl: 168h
  auto_teardown: true
  phase_timeout: 30m
  max_retries: 3
  retry_backoff: exponential

observability:
  metrics_port: 9091
  log_level: info
  tracing:
    enabled: true
    exporter: otlp
```

---

## 12. Directory Structure

```
cluster-provisioner/
├── cmd/
│   ├── server/
│   │   └── main.go              # HTTP/gRPC server
│   └── worker/
│       └── main.go              # Background task worker
├── pkg/
│   ├── api/
│   │   ├── http/                # REST handlers, middleware
│   │   └── grpc/                # gRPC service definitions
│   ├── providers/
│   │   ├── interface.go         # Capability interfaces
│   │   ├── registry.go
│   │   ├── hetzner/
│   │   │   ├── client.go        # Hetzner SDK wrapper
│   │   │   ├── vm.go            # VMLifecycler impl
│   │   │   ├── network.go       # Networker impl
│   │   │   └── rescue.go        # RescueBootable impl
│   │   ├── scaleway/
│   │   │   ├── client.go
│   │   │   ├── vm.go
│   │   │   ├── network.go
│   │   │   └── rescue.go
│   │   └── vultr/
│   │       ├── client.go
│   │       ├── vm.go
│   │       └── network.go
│   ├── clusters/
│   │   ├── interface.go         # Engine interface
│   │   ├── registry.go
│   │   ├── talos/
│   │   │   ├── engine.go        # TalosEngine
│   │   │   ├── plan.go          # Plan generator
│   │   │   ├── install.go       # Rescue mode image flashing
│   │   │   └── config.go        # talosctl config generation
│   │   ├── k3s/
│   │   └── rke2/
│   ├── addons/
│   │   ├── interface.go         # Installer interface
│   │   ├── registry.go
│   │   ├── cilium/
│   │   ├── metallb/
│   │   ├── envoygateway/
│   │   ├── certmanager/
│   │   ├── cloudflaretunnel/
│   │   └── netbird/
│   ├── store/
│   │   ├── interface.go         # Database interface
│   │   └── postgres/            # PostgreSQL implementation
│   ├── tasks/
│   │   ├── interface.go         # Task engine interface
│   │   └── river/               # River implementation
│   ├── events/
│   │   ├── interface.go         # Event bus interface
│   │   └── rabbitmq/            # RabbitMQ implementation
│   ├── secrets/
│   │   ├── interface.go         # Secret store interface
│   │   └── vault/               # HashiCorp Vault implementation
│   ├── reconciler/
│   │   ├── interface.go
│   │   └── worker.go            # Declarative reconciler
│   └── config/
│       └── config.go
├── internal/
│   ├── talosctl/                # Talosctl Go client
│   ├── helm/                    # Helm Go SDK wrapper
│   └── kubectl/                 # K8s Go client
├── migrations/                  # PostgreSQL migrations (golang-migrate)
├── configs/
│   └── config.yaml
├── docker/
│   ├── Dockerfile.server
│   ├── Dockerfile.worker
│   └── docker-compose.yml       # Local dev: PG + RabbitMQ + Vault
├── Makefile
└── README.md
```

---

## 13. Docker Compose (Local Dev)

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: provisioner
      POSTGRES_PASSWORD: provisioner
      POSTGRES_DB: cluster_provisioner
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

  vault:
    image: hashicorp/vault:latest
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: dev-token
    ports:
      - "8200:8200"

  server:
    build:
      context: .
      dockerfile: docker/Dockerfile.server
    ports:
      - "8080:8080"
    environment:
      DB_HOST: postgres
      DB_USER: provisioner
      DB_PASSWORD: provisioner
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672/
      VAULT_ADDR: http://vault:8200
      VAULT_TOKEN: dev-token
    depends_on:
      - postgres
      - rabbitmq
      - vault

  worker:
    build:
      context: .
      dockerfile: docker/Dockerfile.worker
    environment:
      DB_HOST: postgres
      DB_USER: provisioner
      DB_PASSWORD: provisioner
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672/
      VAULT_ADDR: http://vault:8200
      VAULT_TOKEN: dev-token
    depends_on:
      - postgres
      - rabbitmq
      - vault

volumes:
  postgres_data:
  rabbitmq_data:
```

---

## 14. MVP Scope

**Phase 1: Core Infrastructure**
- [ ] PostgreSQL schema + migrations
- [ ] RabbitMQ event bus
- [ ] HashiCorp Vault integration
- [ ] River task engine
- [ ] REST API framework

**Phase 2: Providers**
- [ ] Capability-based provider interface
- [ ] Hetzner provider (extract from poc-hetzner)
- [ ] Scaleway provider (extract from poc-hetzner)
- [ ] Provider registry

**Phase 3: Cluster Engine**
- [ ] Declarative reconciler
- [ ] Talos engine with step plan
- [ ] Step execution + compensation
- [ ] State machine persistence

**Phase 4: Addons**
- [ ] Addon interface with dependencies
- [ ] Cilium installer (Go SDK, not shell)
- [ ] MetalLB installer
- [ ] Addon registry

**Phase 5: Operations**
- [ ] Health checks
- [ ] Auto-teardown (cron job)
- [ ] Events + SSE streaming
- [ ] Kubeconfig endpoint (Vault-secured)

**Phase 6: Production**
- [ ] gRPC API
- [ ] Prometheus metrics
- [ ] Distributed tracing (OpenTelemetry)
- [ ] Web UI (minimal)

---

## 15. Migration from poc-hetzner

| Component | poc-hetzner | cluster-provisioner |
|-----------|-------------|---------------------|
| CLI tool | `talos-poc` binary | REST/gRPC API + Go SDK |
| Provider code | `providers/scaleway/client.go` | `pkg/providers/scaleway/` (capability-based) |
| Talos logic | `cmd_scaleway.go` (monolithic) | `pkg/clusters/talos/` (reconciler + steps) |
| Addon logic | `cmd_install_addons.go` | `pkg/addons/<name>/` (interface + dependencies) |
| State | `keys/state.json` (file) | PostgreSQL + migrations |
| Events | None | RabbitMQ pub/sub |
| Secrets | Env vars + files | HashiCorp Vault |
| Execution | In-process goroutines | River durable tasks |
| Kubeconfig | Local files | Vault-secured, API-served |

---

## 16. Open Questions / Decisions

| # | Question | Status |
|---|----------|--------|
| 1 | Use River or Temporal for task engine? | **River** (lighter, Postgres-native) |
| 2 | Auth/AuthZ: API keys or OAuth? | TBD — start with API keys |
| 3 | Multi-tenancy: org isolation in DB? | TBD — start with single-tenant |
| 4 | Cost tracking: provider billing APIs? | Post-MVP |
| 5 | GitOps: trigger from Git push? | Post-MVP |
| 6 | Node pools: heterogenous workers? | Post-MVP |
| 7 | Auto-scaling: based on metrics? | Post-MVP |
| 8 | Web UI framework? | TBD — likely React + TypeScript |

---

## References

- `poc-hetzner` source: `/home/leadinnovation/Develop/poc-hetzner/`
- Private intranet architecture: `docs/private-intranet-architecture.md`
- River task engine: https://riverqueue.com/
- Temporal workflows: https://temporal.io/
- Cluster API: https://cluster-api.sigs.k8s.io/
- HashiCorp Vault: https://www.vaultproject.io/
