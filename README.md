# Kubernetes Cluster Bootstrapper with Ansible

Ansible playbook to automate Kubernetes cluster installation, supporting **three deployment topologies** in a single repository.

---

## Supported Scenarios

This project supports three Kubernetes deployment topologies, ranging from a simple single-node setup for development purposes to a fully fault-tolerant production-grade HA cluster. Choose the scenario that best fits your infrastructure needs.

| Scenario | Description | Best For |
|---|---|---|
| `base` | Single control-plane, no High Availability | Development / Lab |
| `stacked_etcd` | HA control-plane, etcd stacked with master nodes | Simple Production HA |
| `external_etcd` | HA control-plane, etcd on dedicated nodes | Full Production HA |

![Kubernetes Cluster Bootstrapper with Ansible](docs/topology.png)

---

## Prerequisites

- Ansible >= 2.12
- Python >= 3.8
- SSH access to all nodes
- Nodes running Ubuntu 20.04/22.04 or Debian 11/12

### Install Ansible Dependencies
```bash
ansible-galaxy install -r requirements.yaml
```

---

## Usage

### 1. Select a Scenario

Edit the `cluster_topology` variable in `group_vars/all.yml`:

```yaml
# Choose one of: base | stacked_etcd | external_etcd
cluster_topology: "base"
```

### 2. Configure the Inventory

Edit the inventory file for your chosen scenario:

| Scenario | Inventory File |
|---|---|
| `base` | `inventory/base` |
| `stacked_etcd` | `inventory/stacked_etcd` |
| `external_etcd` | `inventory/external_etcd` |

### 3. Configure Variables

Edit `group_vars/all.yml` to match your environment:

```yaml
# For base scenario
apiserver_advertise_address: "192.168.1.10"

# For stacked_etcd / external_etcd scenarios
lb_ip: "10.0.0.110"
vip_prefix: "24"
keepalived_interface: "eth0"
```

For `stacked_etcd` / `external_etcd` scenarios, also configure `host_vars/`:
- `host_vars/k8s-lb1.yml` — `keepalived_state: MASTER`, `keepalived_priority: 150`
- `host_vars/k8s-lb2.yml` — `keepalived_state: BACKUP`, `keepalived_priority: 100`

### 4. Run the Playbook

```bash
# Base scenario
ansible-playbook site.yml -i inventory/base

# Stacked etcd scenario
ansible-playbook site.yml -i inventory/stacked_etcd

# External etcd scenario
ansible-playbook site.yml -i inventory/external_etcd
```

### Run with Specific Tags

```bash
# Only install common packages and container runtime
ansible-playbook site.yml -i inventory/base --tags "common,container-runtime"

# Only deploy addons
ansible-playbook site.yml -i inventory/base --tags addons
```

---

## Configuration

### Kubernetes

| Variable | Default | Description |
|---|---|---|
| `kubernetes_version` | `1.28.0` | Kubernetes version |
| `cluster_name` | `example-test` | Cluster name |
| `pod_network_cidr` | `10.244.0.0/16` | Pod network CIDR |
| `service_network_cidr` | `10.96.0.0/12` | Service network CIDR |

### CNI Plugin

| Variable | Default | Options |
|---|---|---|
| `cni_plugin` | `calico` | `calico`, `flannel`, `cilium` |

### Container Runtime

| Variable | Default | Options |
|---|---|---|
| `container_runtime` | `containerd` | `containerd`, `docker`, `cri-o` |

### Addons (Available for All Scenarios)

Enable addons in `group_vars/all.yml`:

```yaml
addons:
  helm:
    enabled: true
    version: "v3.17.3"

  metallb:
    enabled: true
    ip_range: "192.168.1.240-192.168.1.250"

  metrics_server:
    enabled: true

  argocd:
    enabled: true

  longhorn:
    enabled: true

  istio:
    enabled: true
    profile: "default"
```

---

## Project Structure

```
k8s-ansible/
├── ansible.cfg
├── site.yml                    # Main playbook (multi-scenario)
├── requirements.yaml
├── docs/
│   └── topology.png            # Cluster topology diagram
├── inventory/
│   ├── base                    # Inventory for base scenario
│   ├── stacked_etcd            # Inventory for stacked HA scenario
│   └── external_etcd           # Inventory for external etcd HA scenario
├── group_vars/
│   └── all.yml                 # All variables + cluster_topology selector
├── host_vars/
│   ├── k8s-lb1.yml             # Keepalived config for LB1 (MASTER)
│   └── k8s-lb2.yml             # Keepalived config for LB2 (BACKUP)
└── roles/
    ├── common/                 # Prerequisite tasks for all nodes
    ├── container-runtime/      # Install containerd/docker/cri-o
    ├── kubernetes/             # Install kubeadm, kubelet, kubectl
    ├── lb/                     # Setup HAProxy + Keepalived (HA only)
    ├── etcd/                   # Setup external etcd cluster (external_etcd only)
    ├── master/                 # Initialize control-plane
    ├── master-join/            # Join additional control-plane nodes (HA only)
    ├── worker/                 # Join worker nodes
    ├── cni/                    # Deploy CNI plugin
    └── addons/                 # Deploy optional addons
```

---

## Roadmap

This project is inspired by [Kubespray](https://github.com/kubernetes-sigs/kubespray) — a production-ready Kubernetes cluster installer using Ansible. However, the long-term vision of this project goes beyond just cluster provisioning.

### 🎯 End Goal

Build a **self-hosted Kubernetes Managed Service** — similar to Amazon EKS or Google GKE — where users can provision and manage fully automated Kubernetes clusters without deep infrastructure expertise.

---

### Current Phase ✅
- [x] Multi-scenario Kubernetes installer (Base, Stacked HA, External ETCD HA)
- [x] Support for multiple CNI plugins (Calico, Flannel, Cilium)
- [x] Support for multiple container runtimes (containerd, Docker, CRI-O)
- [x] Optional addons: Helm, MetalLB, OpenEBS, Longhorn, Metrics Server, ArgoCD, Istio

### Next Phase 🔜 — AI Infrastructure on Kubernetes
- [ ] **GPU Node Support** — Automated NVIDIA GPU Operator installation and configuration
- [ ] **AI-Optimized Playbooks** — Dedicated deployment scenarios for running AI/ML workloads on Kubernetes
- [ ] **Model Serving Stack** — Automated deployment of model serving frameworks (e.g., Triton Inference Server, Ollama, vLLM)
- [ ] **Distributed Training** — Support for Kubeflow and Ray for distributed AI model training
- [ ] **Storage for AI** — High-performance storage class configuration optimized for large model weights

### Future Phase 🔭 — Managed Kubernetes Service
- [ ] **Cluster Lifecycle Management** — Create, upgrade, and delete clusters via a unified control plane
- [ ] **Multi-cluster Support** — Manage multiple clusters from a single interface
- [ ] **Self-service Portal** — Web UI / API for cluster provisioning (like EKS/GKE experience)
- [ ] **Built-in Observability** — Pre-configured monitoring stack (Prometheus, Grafana, Loki)
- [ ] **RBAC & Multi-tenancy** — Tenant isolation and access control for shared clusters
