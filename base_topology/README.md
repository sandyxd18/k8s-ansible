# Kubernetes Cluster Deployment with Ansible

Proyek Ansible ini menyediakan automasi lengkap untuk deployment Kubernetes cluster dengan arsitektur single master dan multi-worker. Proyek ini mendukung konfigurasi yang sangat fleksibel termasuk pemilihan versi Kubernetes, CNI plugin, container registry, dan berbagai addons.

## 🚀 Fitur

- **Arsitektur Fleksibel**: Single master + multiple worker nodes
- **Versi Kubernetes Configurable**: Tentukan versi Kubernetes yang ingin diinstall
- **Multiple CNI Plugins**: Pilih antara Calico, Flannel, Weave, atau Cilium
- **Custom Container Registry**: Konfigurasi registry mirror dan insecure registries
- **Addons Modular**: 
  - NGINX Ingress Controller
  - MetalLB Load Balancer
  - OpenEBS Storage Provider
  - Metrics Server
- **Multi-OS Support**: Mendukung Debian/Ubuntu dan RedHat/CentOS

## 📋 Prerequisites

### Control Machine (tempat menjalankan Ansible)
- Ansible 2.9 atau lebih baru
- Python 3.6+
- SSH access ke semua nodes

### Target Nodes (Master & Workers)
- OS: Ubuntu 20.04/22.04, Debian 11/12, CentOS 7/8, RHEL 7/8
- RAM: Minimal 2GB (Master: 4GB recommended)
- CPU: Minimal 2 cores
- Disk: Minimal 20GB
- Network: Koneksi antar nodes
- User dengan sudo privileges atau root access

## 📁 Struktur Project

```
k8s-ansible/
├── ansible.cfg              # Konfigurasi Ansible
├── inventory                # Inventory file (daftar hosts)
├── site.yml                 # Main playbook
├── group_vars/
│   └── all.yml             # Variabel konfigurasi global
├── roles/
│   ├── common/             # Prerequisites dan system setup
│   ├── container-runtime/  # Containerd installation
│   ├── kubernetes/         # Kubeadm, kubelet, kubectl
│   ├── master/             # Control plane initialization
│   ├── worker/             # Worker node join
│   ├── cni/                # CNI plugin installation
│   └── addons/             # Optional addons
└── README.md
```

## ⚙️ Konfigurasi

### 1. Edit Inventory File

Edit file `inventory` dan sesuaikan dengan IP address nodes Anda:

```ini
[master]
k8s-master ansible_host=192.168.1.10 ansible_user=root

[workers]
k8s-worker1 ansible_host=192.168.1.11 ansible_user=root
k8s-worker2 ansible_host=192.168.1.12 ansible_user=root
k8s-worker3 ansible_host=192.168.1.13 ansible_user=root

[k8s_cluster:children]
master
workers
```

### 2. Konfigurasi Variabel

Edit file `group_vars/all.yml` untuk mengkustomisasi deployment:

#### Versi Kubernetes
```yaml
kubernetes_version: "1.28.0"
kubernetes_version_rhel_package: "1.28.0-0"
```

#### CNI Plugin
Pilih salah satu: `calico`, `flannel`, `weave`, atau `cilium`
```yaml
cni_plugin: "calico"
```

#### Container Registry
```yaml
# Custom registry (opsional)
container_registry: "registry.example.com"

# Registry mirrors
registry_mirrors:
  - "https://mirror.gcr.io"

# Insecure registries
insecure_registries:
  - "registry.example.com:5000"
```

#### Addons
```yaml
addons:
  helm:
    enabled: false
    version: "v3.17.3"
  
  metallb:
    enabled: true
    version: "v0.13.10"
    ip_range: "192.168.1.240-192.168.1.250"
  
  openebs:
    enabled: true
  
  metrics_server:
    enabled: true
    version: "v0.8.0"
```

## 🎯 Cara Penggunaan

### 1. Test Koneksi ke Semua Nodes

```bash
ansible all -m ping
```

```bash
ansible-galaxy collection install -r requirements.yaml
sudo apt install python3-pip
pip3 install kubernetes
```

### 2. Deploy Kubernetes Cluster

```bash
ansible-playbook -i inventory site.yml
```

### 3. Verify Deployment

Setelah deployment selesai, login ke master node dan jalankan:

```bash
kubectl get nodes
kubectl get pods -A
```

### 4. Deploy Hanya Addons Tertentu

Jika cluster sudah berjalan dan Anda hanya ingin menambah addons:

```bash
ansible-playbook -i inventory site.yml --tags addons
```

## 🔧 Opsi Deployment Lanjutan

### Deploy dengan Specific Tags

```bash
# Hanya setup prerequisites
ansible-playbook -i inventory site.yml --tags common

# Hanya install container runtime
ansible-playbook -i inventory site.yml --tags container-runtime

# Hanya install Kubernetes packages
ansible-playbook -i inventory site.yml --tags kubernetes
```

### Limit Execution ke Host Tertentu

```bash
# Hanya jalankan di master
ansible-playbook -i inventory site.yml --limit master

# Hanya jalankan di worker tertentu
ansible-playbook -i inventory site.yml --limit k8s-worker1
```

### Dry Run (Check Mode)

```bash
ansible-playbook -i inventory site.yml --check
```

## 📝 Contoh Konfigurasi

### Contoh 1: Production Setup dengan Calico

```yaml
kubernetes_version: "1.28.0"
cni_plugin: "calico"
pod_network_cidr: "10.244.0.0/16"

addons:
  nginx_ingress:
    enabled: true
  metallb:
    enabled: true
    ip_range: "192.168.1.240-192.168.1.250"
  openebs:
    enabled: true
  metrics_server:
    enabled: true
```

### Contoh 2: Development Setup dengan Flannel

```yaml
kubernetes_version: "1.27.0"
cni_plugin: "flannel"
pod_network_cidr: "10.244.0.0/16"

addons:
  nginx_ingress:
    enabled: false
  metallb:
    enabled: false
  openebs:
    enabled: false
  metrics_server:
    enabled: true
```

### Contoh 3: Setup dengan Private Registry

```yaml
kubernetes_version: "1.28.0"
cni_plugin: "calico"

container_registry: "harbor.company.com"
insecure_registries:
  - "harbor.company.com"

registry_mirrors:
  - "https://harbor.company.com"
```

## 🔍 Troubleshooting

### Cluster Initialization Gagal

Jika initialization gagal, reset cluster di master node:

```bash
kubeadm reset -f
rm -rf /etc/kubernetes/
rm -rf /var/lib/etcd/
```

Kemudian jalankan ulang playbook.

### Worker Node Tidak Bisa Join

1. Pastikan firewall mengizinkan koneksi
2. Cek join token masih valid:
   ```bash
   kubeadm token list
   ```
3. Generate token baru jika perlu:
   ```bash
   kubeadm token create --print-join-command
   ```

### CNI Plugin Tidak Berfungsi

Cek status pods CNI:
```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system <cni-pod-name>
```

### Addons Tidak Terinstall

Pastikan addons diaktifkan di `group_vars/all.yml` dan jalankan:
```bash
ansible-playbook -i inventory site.yml --tags addons
```

## 🛡️ Security Considerations

1. **SSH Keys**: Gunakan SSH key authentication daripada password
2. **Firewall**: Konfigurasi firewall untuk membatasi akses
3. **RBAC**: Implementasikan RBAC policies setelah cluster berjalan
4. **Network Policies**: Gunakan network policies untuk isolasi pods
5. **Registry Security**: Gunakan TLS untuk container registries di production

## 📚 Referensi

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Ansible Documentation](https://docs.ansible.com/)
- [Calico Documentation](https://docs.projectcalico.org/)
- [Flannel Documentation](https://github.com/flannel-io/flannel)
- [MetalLB Documentation](https://metallb.universe.tf/)
- [OpenEBS Documentation](https://openebs.io/docs)

## 📄 License

MIT License - Silakan gunakan dan modifikasi sesuai kebutuhan.

## 🤝 Contributing

Kontribusi sangat diterima! Silakan buat pull request atau issue untuk improvement.

## ✨ Author

Created with ❤️ for Kubernetes automation
