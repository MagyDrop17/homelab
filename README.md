# Homelab

3-node Kubernetes cluster running on Raspberry Pi 5 hardware, managed via
GitOps. This repo contains the Ansible playbooks, Flux manifests, and
documentation for building and operating the cluster.

## Hardware

- 3× Raspberry Pi 5 (16 GB RAM each)
- 1 TB NVMe per node (data) + 117 GB SD card per node (OS)
- Flat home LAN (Adamo router)

## Network

| Host          | IP              | Role                          |
|---------------|-----------------|-------------------------------|
| rp-node-1     | 192.168.1.230   | control plane + worker        |
| rp-node-2     | 192.168.1.231   | control plane + worker        |
| rp-node-3     | 192.168.1.232   | control plane + worker        |
| cluster VIP   | 192.168.1.240   | floating API endpoint         |

## Stack

- **OS:** Ubuntu Server 24.04 LTS (ARM64)
- **Kubernetes:** k3s, HA with embedded etcd (3 control planes)
- **VIP:** kube-vip (ARP mode)
- **Storage:** Longhorn (replicated block storage on NVMes)
- **Load balancer:** MetalLB
- **Ingress:** ingress-nginx
- **TLS:** cert-manager
- **GitOps:** FluxCD
- **Git server:** Forgejo (self-hosted, eventually hosts this repo)
- **Identity:** Keycloak + PrivacyIDEA
- **VPN:** NetBird
- **Observability:** Prometheus + Grafana + Loki
- **Backups:** Velero

## Repo layout

    homelab/
    ├── README.md           ← you are here
    ├── docs/               ← concepts and phase notes
    ├── ansible/            ← OS provisioning (Phase 1)
    └── clusters/           ← Flux-managed manifests (Phase 4+)

## Phases

0. ✅ Workstation prep (WSL2 + toolchain)
1. ✅ OS hardening with Ansible
2. ✅ k3s install (HA)
3. ✅ Platform layer (MetalLB, ingress, cert-manager, Longhorn)
4. ✅ Flux bootstrap
5. Forgejo + migrate Flux source
6. Identity, VPN, apps
7. Observability + backups
# Flux source: Forgejo (self-hosted)
# Flux source: Forgejo (self-hosted)
