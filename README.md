# Cluster Provisioner

A Go service for provisioning Kubernetes clusters on multiple cloud providers.

## Goal

Provider-agnostic cluster provisioning with pluggable:
- **Cloud providers**: Hetzner, Scaleway, Vultr, AWS, GCP, etc.
- **Cluster types**: Talos, k3s, RKE2
- **Addons**: CNI, ingress, VPN, storage, monitoring

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full design plan.

## Inspiration

Built from lessons learned in [poc-hetzner](https://github.com/mtnguyen01/poc-hetzner) — a CLI tool that deploys Talos clusters on Hetzner/Scaleway/Vultr with VPN, CNI, and ingress.

## Status

🚧 **Planning phase** — architecture review in progress.
