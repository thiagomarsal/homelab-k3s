# Homelab k3s HA Cluster

Ansible automation for a 3-node HA k3s cluster with Rancher, running on Proxmox.

## Architecture

| VM | Host | IP | Role |
|---|---|---|---|
| k3s-master-1 | pve01 (N95) | 192.168.1.50 | server + worker (8G) |
| k3s-master-2 | pve02 (N95) | 192.168.1.51 | server + worker (10G) |
| k3s-master-3 | pve03 (i5-6500T) | 192.168.1.52 | server + worker (6G) |
| k3s-worker-1 | pve04 (Xeon E5-2680) | 192.168.1.53 | agent, on-demand (128G) |

- **API VIP:** 192.168.1.60 (kube-vip)
- **MetalLB range:** 192.168.1.61–199
- **OS:** Debian 12 (Bookworm)

## Prerequisites

- 4 Debian 12 VMs created via cloud-init on Proxmox
- SSH key (`~/.ssh/id_k3s`) with access to all VMs as `tfarias`
- Ansible installed on your workstation

## Quick Start

```bash
# 1. Generate a cluster token
openssl rand -hex 32

# 2. Update the token in group_vars/all.yml (k3s_token)

# 3. Run the full install
ansible-playbook playbooks/site.yml

# 4. Use the cluster
export KUBECONFIG=./kubeconfig.yml
kubectl get nodes
```

## Playbooks

| Playbook | Description |
|---|---|
| `playbooks/site.yml` | Full install: OS prep → k3s HA → Rancher |
| `playbooks/reset.yml` | Tear down k3s completely (destructive!) |

## Cluster Services

Installed automatically by `site.yml`:

- **kube-vip** — API server HA via virtual IP (ARP/L2)
- **MetalLB** — LoadBalancer service IPs
- **Traefik** — Ingress controller (custom install, not bundled)
- **cert-manager** — TLS certificate automation
- **Rancher** — Cluster management UI

## Secrets

The `k3s_token` in `group_vars/all.yml` should be encrypted:

```bash
ansible-vault encrypt_string 'your-token-here' --name 'k3s_token'
```

## Reset

To completely tear down and start fresh:

```bash
ansible-playbook playbooks/reset.yml
```
