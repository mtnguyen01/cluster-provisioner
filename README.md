# OpenClaw Platform

A suite of tools for managing infrastructure and AI agents at scale.

## Products

### 1. Cluster Provisioner
A Go service for provisioning Kubernetes clusters on multiple cloud providers.

- **Backbone:** PostgreSQL + RabbitMQ
- **Purpose:** Deploy Talos/k3s/RKE2 clusters on Hetzner, Scaleway, Vultr
- **Features:** Multi-provider, addon management, durable task execution

[See CLUSTER_PROVISIONER.md](CLUSTER_PROVISIONER.md)

### 2. Agent Management Platform
A Go service for managing fleets of OpenClaw agents.

- **Backbone:** PostgreSQL + RabbitMQ
- **Purpose:** Control plane for OpenClaw agents across teams
- **Features:** Scoped permissions, health checks, audit trails, approval gates

[See AGENT_PLATFORM.md](AGENT_PLATFORM.md)

## Shared Infrastructure

Both products share:
- PostgreSQL for state
- RabbitMQ for events
- HashiCorp Vault for secrets
- River for durable task execution
- Go-based service architecture

## Status

🚧 **Architecture phase** — both designs complete, implementation starting.

## Repos

- Platform docs: https://github.com/mtnguyen01/cluster-provisioner
- OpenClaw: https://github.com/openclaw/openclaw
