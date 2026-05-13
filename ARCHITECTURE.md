# Cluster Provisioner Service — Architecture Plan

**Goal:** A Go service that provisions different cluster types on different hosting providers through a unified API.

**Inspiration:** `poc-hetzner` (Talos on Hetzner/Scaleway/Vultr with VPN, CNI, ingress)

**Date:** 2026-05-13

---

## 1. Service Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Cluster Provisioner Service                      │
│                          (Go + gRPC/HTTP)                           │
└─────────────────────────────────────────────────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │   Hetzner    │    │   Scaleway   │    │    Vultr     │
   │  Provider    │    │  Provider    │    │  Provider    │
   └──────────────┘    └──────────────┘    └──────────────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │   Cluster Types     │
                    │  • Talos K8s        │
                    │  • k3s              │
                    │  • RKE2             │
                    │  • plain Docker     │
                    └─────────────────────┘
```

**Core idea:** Provider-agnostic cluster provisioning with pluggable:
- Cloud providers (Hetzner, Scaleway, Vultr, AWS, GCP, etc.)
- Cluster types (Talos, k3s, RKE2, etc.)
- Addons (CNI, ingress, VPN, storage, monitoring)

---

## 2. Architecture

### 2.1 Layers

```
┌─────────────────────────────────────────┐
│           API Layer (gRPC/REST)         │
│  CreateCluster, GetCluster, DeleteCluster│
├─────────────────────────────────────────┤
│         Orchestration Layer             │
│  State machine, retry logic, events     │
├─────────────────────────────────────────┤
│         Provider Interface              │
│  CreateVM, DeleteVM, CreateNetwork, etc │
├─────────────────────────────────────────┤
│         Cluster Interface               │
│  GenerateConfig, Bootstrap, Validate    │
├─────────────────────────────────────────┤
│         Addon Interface                 │
│  InstallCNI, InstallIngress, InstallVPN │
├─────────────────────────────────────────┤
│         Cloud SDKs                      │
│  Hetzner SDK, Scaleway SDK, Vultr SDK   │
└─────────────────────────────────────────┘
```

### 2.2 Key Components

| Component | Responsibility |
|-----------|---------------|
| **API Server** | Accepts provision requests, returns cluster status |
| **State Store** | Persists cluster state (etcd/SQLite/Postgres) |
| **Worker Pool** | Async execution of provisioning tasks |
| **Provider Registry** | Discovers and loads provider plugins |
| **Cluster Engine** | Orchestrates bootstrap sequence per cluster type |
| **Addon Manager** | Installs/updates/removes cluster addons |
| **Event Bus** | Publishes provisioning events (Webhook/SSE) |

---

## 3. Data Model

### 3.1 Cluster

```go
type Cluster struct {
    ID        string    // UUID
    Name      string    // user-provided
    Provider  string    // "hetzner", "scaleway", "vultr"
    Region    string    // "fsn1", "nl-ams-1", "ams"
    Type      string    // "talos", "k3s", "rke2"
    
    // Topology
    ControlPlanes int
    Workers       int
    
    // Network
    VPCCIDR       string
    PrivateIPs    bool    // use private IPs for K8s API
    
    // State
    Phase     ClusterPhase  // Pending, Provisioning, Ready, Deleting, Failed
    State     ClusterState  // serialized provider-specific state
    
    // Endpoints
    Kubeconfig    string    // base64 encoded
    PublicEndpoint string
    PrivateEndpoint string
    
    // Timestamps
    CreatedAt time.Time
    UpdatedAt time.Time
    ExpiresAt *time.Time  // auto-teardown
    
    // Addons
    Addons []AddonSpec
}

type ClusterPhase string
const (
    PhasePending       ClusterPhase = "Pending"
    PhaseProvisioning  ClusterPhase = "Provisioning"
    PhaseReady         ClusterPhase = "Ready"
    PhaseDeleting      ClusterPhase = "Deleting"
    PhaseFailed        ClusterPhase = "Failed"
)
```

### 3.2 Node

```go
type Node struct {
    ID         string
    ClusterID  string
    Name       string
    Role       NodeRole  // ControlPlane, Worker, VPN
    
    // IPs
    PublicIP   string
    PrivateIP  string
    VPNIP      string  // Netbird/Tailscale IP
    
    // Provider
    ProviderID string  // cloud provider VM ID
    
    // State
    Status     NodeStatus  // Creating, Running, Failed, Deleted
}

type NodeRole string
const (
    RoleControlPlane NodeRole = "control-plane"
    RoleWorker       NodeRole = "worker"
    RoleVPN          NodeRole = "vpn"
)
```

### 3.3 Addon

```go
type AddonSpec struct {
    Name    string            // "cilium", "metallb", "envoy-gateway"
    Version string            // "v1.19.3"
    Config  map[string]string // addon-specific config
    Enabled bool
}
```

---

## 4. Provider Interface

### 4.1 Core Interface

```go
package providers

// Provider is the interface for cloud providers
type Provider interface {
    // Identity
    Name() string
    
    // VM Lifecycle
    CreateVM(ctx context.Context, opts VMOptions) (*VM, error)
    DeleteVM(ctx context.Context, id string) error
    GetVM(ctx context.Context, id string) (*VM, error)
    ListVMs(ctx context.Context, tags map[string]string) ([]VM, error)
    
    // Network
    CreateNetwork(ctx context.Context, opts NetworkOptions) (*Network, error)
    DeleteNetwork(ctx context.Context, id string) error
    AttachNetwork(ctx context.Context, vmID, networkID string) error
    
    // Storage
    CreateVolume(ctx context.Context, opts VolumeOptions) (*Volume, error)
    AttachVolume(ctx context.Context, vmID, volumeID string) error
    
    // Load Balancer
    CreateLoadBalancer(ctx context.Context, opts LBOptions) (*LoadBalancer, error)
    DeleteLoadBalancer(ctx context.Context, id string) error
    
    // SSH Keys
    UploadSSHKey(ctx context.Context, name, publicKey string) (*SSHKey, error)
    DeleteSSHKey(ctx context.Context, id string) error
    
    // Rescue / Recovery
    EnableRescueMode(ctx context.Context, vmID string) error
    DisableRescueMode(ctx context.Context, vmID string) error
    
    // Boot / Image
    BootFromImage(ctx context.Context, vmID, imageID string) error
    GetImage(ctx context.Context, name, architecture string) (*Image, error)
}
```

### 4.2 VM Options

```go
type VMOptions struct {
    Name       string
    ServerType string  // "small", "medium", "large" or provider-specific
    Image      string  // OS image name
    Location   string  // region/zone
    
    Network    NetworkAttachment
    SSHKeyIDs  []string
    
    Labels     map[string]string
    UserData   string  // cloud-init
    
    // Talos-specific
    TalosVersion string
}
```

### 4.3 Provider Registry

```go
type Registry struct {
    providers map[string]Provider
}

func (r *Registry) Register(name string, p Provider) {
    r.providers[name] = p
}

func (r *Registry) Get(name string) (Provider, error) {
    p, ok := r.providers[name]
    if !ok {
        return nil, fmt.Errorf("provider %q not registered", name)
    }
    return p, nil
}

func (r *Registry) List() []string {
    // return registered provider names
}
```

---

## 5. Cluster Type Interface

### 5.1 Core Interface

```go
package clusters

// Engine provisions a specific cluster type
type Engine interface {
    // Identity
    Name() string
    Description() string
    
    // Validation
    ValidateSpec(spec ClusterSpec) error
    
    // Bootstrap sequence
    Bootstrap(ctx context.Context, cluster *Cluster, provider providers.Provider) error
    
    // Health checks
    HealthCheck(ctx context.Context, cluster *Cluster) (HealthStatus, error)
    
    // Kubeconfig
    GetKubeconfig(ctx context.Context, cluster *Cluster) ([]byte, error)
    
    // Cleanup
    Destroy(ctx context.Context, cluster *Cluster, provider providers.Provider) error
}
```

### 5.2 Talos Engine (reference implementation)

```go
type TalosEngine struct {
    talosctlPath string
}

func (e *TalosEngine) Bootstrap(ctx context.Context, cluster *Cluster, p providers.Provider) error {
    // Phase 1: Create VMs (Debian base)
    // Phase 2: Rescue mode → install Talos via dd
    // Phase 3: Generate Talos config
    // Phase 4: Apply config with --insecure
    // Phase 5: Bootstrap etcd
    // Phase 6: Wait for K8s API
    // Phase 7: Generate kubeconfig
    return nil
}
```

### 5.3 Cluster Spec

```go
type ClusterSpec struct {
    Name      string
    Provider  string
    Region    string
    Type      string
    
    ControlPlanes int
    Workers       int
    
    // Networking
    VPCCIDR     string
    UsePrivateIP bool
    VPNType     string  // "netbird", "tailscale", "headscale", ""
    
    // Addons to install
    Addons []AddonSpec
}
```

---

## 6. Addon Interface

### 6.1 Core Interface

```go
package addons

// Installer provisions a cluster addon
type Installer interface {
    Name() string
    Description() string
    
    // Check if addon is already installed
    IsInstalled(ctx context.Context, cluster *Cluster) (bool, error)
    
    // Install the addon
    Install(ctx context.Context, cluster *Cluster, config map[string]string) error
    
    // Uninstall
    Uninstall(ctx context.Context, cluster *Cluster) error
    
    // Health check
    HealthCheck(ctx context.Context, cluster *Cluster) error
}
```

### 6.2 Addon Registry

```go
type Registry struct {
    installers map[string]Installer
}

// Built-in addons:
// - cilium
// - metallb
// - envoy-gateway
// - cert-manager
// - cloudflare-tunnel
// - netbird
// - scaleway-ccm
// - scaleway-csi
```

### 6.3 Example: Cilium Installer

```go
type CiliumInstaller struct{}

func (i *CiliumInstaller) Install(ctx context.Context, cluster *Cluster, config map[string]string) error {
    // helm repo add cilium
    // helm install cilium cilium/cilium --version v1.19.3
    // kubectl wait --for=condition=ready
    return nil
}
```

---

## 7. Provisioning Flow (State Machine)

```
[Pending]
   │
   ▼
[Creating Network] ──► [Network Ready]
   │
   ▼
[Creating VMs] ──► [VMs Running]
   │
   ▼
[Installing OS / Bootstrapping] ──► [Nodes Ready]
   │
   ▼
[Installing CNI] ──► [CNI Ready]
   │
   ▼
[Installing Addons] ──► [Addons Ready]
   │
   ▼
[Installing VPN] ──► [VPN Ready] (optional)
   │
   ▼
[Installing Ingress] ──► [Ingress Ready]
   │
   ▼
[Installing Apps] ──► [Apps Ready] (optional)
   │
   ▼
[Ready]
```

Each phase:
- Has a timeout
- Is retryable
- Publishes events
- Updates state in store
- Can be rolled back on failure

---

## 8. API Design

### 8.1 REST API (v1)

```
POST   /v1/clusters              → Create cluster
GET    /v1/clusters              → List clusters
GET    /v1/clusters/:id          → Get cluster status
DELETE /v1/clusters/:id          → Delete cluster
POST   /v1/clusters/:id/addons  → Install addon
GET    /v1/clusters/:id/kubeconfig → Download kubeconfig
GET    /v1/clusters/:id/nodes    → List nodes
POST   /v1/clusters/:id/upgrade  → Upgrade cluster
```

### 8.2 Create Cluster Request

```json
{
  "name": "prod-eu",
  "provider": "scaleway",
  "region": "nl-ams-1",
  "type": "talos",
  "control_planes": 1,
  "workers": 2,
  "vpc_cidr": "10.0.0.0/22",
  "use_private_ip": true,
  "vpn_type": "netbird",
  "addons": [
    {"name": "cilium", "version": "v1.19.3"},
    {"name": "metallb"},
    {"name": "envoy-gateway"},
    {"name": "cert-manager"},
    {"name": "cloudflare-tunnel", "config": {"domain": "hello.hyantechnology.dev"}}
  ],
  "ttl_hours": 24
}
```

### 8.3 Cluster Status Response

```json
{
  "id": "cluster-abc123",
  "name": "prod-eu",
  "phase": "Ready",
  "provider": "scaleway",
  "region": "nl-ams-1",
  "type": "talos",
  "control_planes": 1,
  "workers": 2,
  "nodes": [
    {
      "name": "prod-eu-cp1",
      "role": "control-plane",
      "public_ip": "51.15.69.129",
      "private_ip": "10.0.0.2",
      "status": "Running"
    }
  ],
  "endpoints": {
    "public": "https://51.15.69.129:6443",
    "private": "https://10.0.0.2:6443"
  },
  "addons": [
    {"name": "cilium", "status": "Ready"},
    {"name": "envoy-gateway", "status": "Ready"}
  ],
  "created_at": "2026-05-13T07:00:00Z",
  "expires_at": "2026-05-14T07:00:00Z"
}
```

---

## 9. Configuration

### 9.1 Service Config

```yaml
# config.yaml
server:
  http_port: 8080
  grpc_port: 9090

state_store:
  type: sqlite  # sqlite, postgres, etcd
  path: /var/lib/cluster-provisioner/state.db

providers:
  hetzner:
    token: ${HCLOUD_TOKEN}
  scaleway:
    access_key: ${SCALEWAY_ACCESS_KEY_ID}
    secret: ${SCALEWAY_SECRET}
    project_id: ${SCW_PROJECT_ID}
  vultr:
    api_key: ${VULTR_API_KEY}

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
  max_ttl: 168h  # 7 days
  auto_teardown: true
  worker_pool_size: 5
  phase_timeout: 30m

observability:
  metrics_port: 9091
  log_level: info
```

---

## 10. Directory Structure

```
cluster-provisioner/
├── cmd/
│   └── server/
│       └── main.go
├── pkg/
│   ├── api/
│   │   ├── http/          # REST handlers
│   │   └── grpc/          # gRPC service
│   ├── providers/
│   │   ├── interface.go   # Provider interface
│   │   ├── registry.go    # Provider registry
│   │   ├── hetzner/
│   │   ├── scaleway/
│   │   ├── vultr/
│   │   └── factory.go     # Auto-detect provider
│   ├── clusters/
│   │   ├── interface.go   # Cluster engine interface
│   │   ├── registry.go
│   │   ├── talos/
│   │   ├── k3s/
│   │   └── rke2/
│   ├── addons/
│   │   ├── interface.go   # Addon installer interface
│   │   ├── registry.go
│   │   ├── cilium/
│   │   ├── metallb/
│   │   ├── envoygateway/
│   │   ├── certmanager/
│   │   ├── cloudflaretunnel/
│   │   └── netbird/
│   ├── state/
│   │   ├── interface.go   # State store interface
│   │   └── sqlite/
│   ├── orchestrator/
│   │   ├── worker.go      # Async task worker
│   │   ├── statemachine.go
│   │   └── events.go
│   └── config/
│       └── config.go
├── internal/
│   ├── talosctl/          # Talosctl version management
│   ├── helm/              # Helm wrapper
│   └── kubectl/           # Kubectl wrapper
├── migrations/            # DB migrations
├── configs/
│   └── config.yaml
├── Dockerfile
├── Makefile
└── README.md
```

---

## 11. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Interface-based providers** | Easy to add new cloud providers |
| **Interface-based cluster engines** | Easy to add k3s, RKE2, etc. |
| **Interface-based addons** | Easy to add new cluster components |
| **SQLite default** | Zero-dependency for single-node deployment |
| **Async worker pool** | Provisioning is long-running, shouldn't block API |
| **State machine** | Clear phases, retry logic, observability |
| **Auto-TEARDOWN** | Prevents billing surprises |
| **gRPC + REST** | gRPC for internal, REST for external/clients |
| **Base64 kubeconfig in API** | Easy consumption by CI/CD tools |

---

## 12. MVP Scope

**Phase 1:** Core service
- [ ] SQLite state store
- [ ] REST API
- [ ] Hetzner provider (reuse from poc-hetzner)
- [ ] Talos engine (reuse from poc-hetzner)
- [ ] Async worker pool
- [ ] State machine

**Phase 2:** More providers
- [ ] Scaleway provider (migrate from poc-hetzner)
- [ ] Vultr provider (migrate from poc-hetzner)

**Phase 3:** Addons
- [ ] Cilium installer
- [ ] MetalLB installer
- [ ] Envoy Gateway installer

**Phase 4:** Advanced features
- [ ] Cloudflare Tunnel addon
- [ ] Netbird VPN addon
- [ ] cert-manager addon
- [ ] Auto-teardown
- [ ] Event webhooks

**Phase 5:** Polish
- [ ] gRPC API
- [ ] Metrics (Prometheus)
- [ ] Web UI
- [ ] k3s/RKE2 engines

---

## 13. Migration from poc-hetzner

| Component | poc-hetzner | cluster-provisioner |
|-----------|-------------|---------------------|
| CLI tool | `talos-poc` binary | `cluster-provisioner` server + CLI client |
| Provider code | `providers/scaleway/client.go` | `pkg/providers/scaleway/` (extract, clean up) |
| Talos logic | `cmd_scaleway.go` | `pkg/clusters/talos/` (extract phases) |
| Addon logic | `cmd_install_addons.go` | `pkg/addons/<name>/` (split per addon) |
| State | `keys/state.json` | SQLite/Postgres with migrations |
| Config | Hardcoded + env vars | `config.yaml` + env var substitution |

---

## 14. Open Questions

1. **Auth/AuthZ:** API key? OAuth? mTLS?
2. **Multi-tenancy:** One service per org, or namespace isolation?
3. **Cost tracking:** Integrate with provider billing APIs?
4. **GitOps integration:** Trigger provisioning from Git push?
5. **Node pools:** Support heterogenous worker sizes?
6. **Auto-scaling:** Scale workers up/down based on metrics?

---

## References

- `poc-hetzner` source: `/home/leadinnovation/Develop/poc-hetzner/`
- Scaleway provider: `providers/scaleway/client.go`
- Talos commands: `cmd_scaleway.go`
- Addon commands: `cmd_install_addons.go`
- Private intranet architecture: `docs/private-intranet-architecture.md`
