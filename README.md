# Cluster Provisioner

A production Go service for provisioning Kubernetes clusters on multiple cloud providers.

## Architecture

- **PostgreSQL** — primary state store (clusters, nodes, resources, tasks)
- **RabbitMQ** — event bus and pub/sub for provisioning events
- **HashiCorp Vault** — secret management (kubeconfig, cloud API tokens, SSH keys)
- **River** — durable task execution engine for long-running provisioning workflows
- **Go SDKs** — direct provider integration (no shelling out to `talosctl` / `helm` / `kubectl`)

## Supported Providers

| Provider | Status | Capabilities |
|----------|--------|-------------|
| Hetzner | 🚧 Planned | VMs, Networks, Rescue Mode, Firewalls, Volumes, LBs |
| Scaleway | 🚧 Planned | VMs, VPC, Rescue Mode, Firewalls, CCM, CSI |
| Vultr | 🚧 Planned | VMs, VPC, Custom Images, Firewalls |

## Supported Cluster Types

| Type | Status | Description |
|------|--------|-------------|
| Talos | 🚧 Planned | Immutable Linux + Kubernetes |
| k3s | 🔮 Future | Lightweight Kubernetes |
| RKE2 | 🔮 Future | Rancher Kubernetes Engine 2 |

## Supported Addons

| Addon | Dependencies | Status |
|-------|-------------|--------|
| Cilium | — | 🚧 Planned |
| MetalLB | Cilium | 🚧 Planned |
| Scaleway CCM | — | 🚧 Planned |
| Envoy Gateway | Cilium | 🚧 Planned |
| cert-manager | — | 🚧 Planned |
| Cloudflare Tunnel | Envoy Gateway + cert-manager | 🚧 Planned |
| Netbird VPN | — | 🚧 Planned |

## API

```
POST   /v1/clusters              → Create cluster (202 Accepted)
GET    /v1/clusters              → List clusters
GET    /v1/clusters/:id          → Get cluster status
DELETE /v1/clusters/:id          → Delete cluster
GET    /v1/clusters/:id/events  → SSE stream of provisioning events
GET    /v1/clusters/:id/kubeconfig → Download kubeconfig
```

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full design.

## Local Development

```bash
# Start dependencies
docker compose -f docker/docker-compose.yml up -d

# Run migrations
make migrate-up

# Seed Vault secrets
make vault-seed

# Start server
make server

# Start worker (in another terminal)
make worker
```

## Inspiration

Built from lessons learned in [poc-hetzner](https://github.com/mtnguyen01/poc-hetzner) — a CLI tool that deploys Talos clusters with VPN, CNI, and ingress on Hetzner/Scaleway/Vultr.

## Status

🚧 **Architecture phase** — design complete, implementation starting.
