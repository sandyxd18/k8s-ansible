# Kubernetes Ansible Installer

Ansible playbook untuk instalasi Kubernetes yang mendukung **3 skenario topologi** dalam satu repository.

---

## Skenario yang Didukung

| Skenario | Deskripsi | Cocok Untuk |
|---|---|---|
| `base` | Single control-plane, tanpa High Availability | Development / Lab |
| `stacked_etcd` | HA control-plane, etcd stack bersama master | Production HA sederhana |
| `external_etcd` | HA control-plane, etcd di node terpisah | Production HA penuh |

### Topologi `base`
```
[masters]          [workers]
  master  ──────── worker1
                   worker2
                   worker3
```

### Topologi `stacked_etcd`
```
[load_balancers]   [masters]          [workers]
   lb1 (VIP) ─┬── master1 (etcd)     worker1
   lb2        └── master2 (etcd) ─── worker2
```

### Topologi `external_etcd`
```
[load_balancers]   [masters]     [etcd]      [workers]
   lb1 (VIP) ─┬── master1       etcd1        worker1
   lb2        └── master2 ───── etcd2 ─────  worker2
                                etcd3
```

---

## Prasyarat

- Ansible >= 2.12
- Python >= 3.8
- SSH akses ke semua node
- Node berbasis Ubuntu 20.04/22.04 atau Debian 11/12

### Install Ansible Dependencies
```bash
ansible-galaxy install -r requirements.yaml
```

---

## Cara Penggunaan

### 1. Pilih Skenario

Edit variabel `cluster_topology` di `group_vars/all.yml`:

```yaml
# Pilih salah satu: base | stacked_etcd | external_etcd
cluster_topology: "base"
```

### 2. Sesuaikan Inventory

Edit file inventory sesuai skenario yang dipilih:

| Skenario | File Inventory |
|---|---|
| `base` | `inventory/base` |
| `stacked_etcd` | `inventory/stacked_etcd` |
| `external_etcd` | `inventory/external_etcd` |

### 3. Sesuaikan Variabel

Edit `group_vars/all.yml` sesuai kebutuhan:

```yaml
# Untuk skenario base
apiserver_advertise_address: "192.168.1.10"

# Untuk skenario stacked_etcd / external_etcd
lb_ip: "10.0.0.110"
vip_prefix: "24"
keepalived_interface: "eth0"
```

Untuk skenario `stacked_etcd` / `external_etcd`, sesuaikan juga `host_vars/`:
- `host_vars/k8s-lb1.yml` — `keepalived_state: MASTER`, `keepalived_priority: 150`
- `host_vars/k8s-lb2.yml` — `keepalived_state: BACKUP`, `keepalived_priority: 100`

### 4. Jalankan Playbook

```bash
# Skenario base
ansible-playbook site.yml -i inventory/base

# Skenario stacked_etcd
ansible-playbook site.yml -i inventory/stacked_etcd

# Skenario external_etcd
ansible-playbook site.yml -i inventory/external_etcd
```

### Menjalankan Sebagian (Tags)

```bash
# Hanya install common + container runtime
ansible-playbook site.yml -i inventory/base --tags "common,container-runtime"

# Hanya deploy addons
ansible-playbook site.yml -i inventory/base --tags addons
```

---

## Konfigurasi

### Kubernetes

| Variabel | Default | Deskripsi |
|---|---|---|
| `kubernetes_version` | `1.28.0` | Versi Kubernetes |
| `cluster_name` | `example-test` | Nama cluster |
| `pod_network_cidr` | `10.244.0.0/16` | CIDR jaringan Pod |
| `service_network_cidr` | `10.96.0.0/12` | CIDR Service |

### CNI Plugin

| Variabel | Default | Pilihan |
|---|---|---|
| `cni_plugin` | `calico` | `calico`, `flannel`, `cilium` |

### Container Runtime

| Variabel | Default | Pilihan |
|---|---|---|
| `container_runtime` | `containerd` | `containerd`, `docker`, `cri-o` |

### Addons (Berlaku untuk Semua Skenario)

Aktifkan addon di `group_vars/all.yml`:

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

## Struktur Proyek

```
k8s-ansible/
├── ansible.cfg
├── site.yml                    # Main playbook (multi-skenario)
├── requirements.yaml
├── inventory/
│   ├── base                    # Inventory untuk skenario base
│   ├── stacked_etcd            # Inventory untuk skenario stacked HA
│   └── external_etcd           # Inventory untuk skenario external etcd HA
├── group_vars/
│   └── all.yml                 # Semua variabel + cluster_topology selector
├── host_vars/
│   ├── k8s-lb1.yml             # Keepalived config untuk LB1 (MASTER)
│   └── k8s-lb2.yml             # Keepalived config untuk LB2 (BACKUP)
└── roles/
    ├── common/                 # Prerequisite semua node
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
