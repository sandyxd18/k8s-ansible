# Base Topology Kubernetes Cluster Deployment

This Ansible project provides full automation for deploying a Kubernetes cluster with a single-master and multi-worker architecture. The project supports highly flexible configuration including selection of Kubernetes version, CNI plugin, and various addons.

## 🚀 Features

- **Flexible Architecture**: Single master + multiple worker nodes
- **Configurable Kubernetes Version**: Specify the Kubernetes version to install
- **Multiple CNI Plugins**: Choose between Calico, Flannel, Weave, or Cilium
- **Modular Addons**:
  - MetalLB Load Balancer
  - OpenEBS Storage Provider
  - Metrics Server
- **Multi-OS Support**: Supports Debian/Ubuntu and RedHat/CentOS

## 📋 Prerequisites

### Control Machine (Ansible Machine)
- Ansible 2.9 or newer
- Python 3.6+
- SSH access to all nodes

### Target Nodes (Master & Workers)
- OS: Ubuntu 22.04 or newer, CentOS 8 or newer
- RAM: Minimum 2GB (Master: 4GB recommended)
- CPU: Minimum 2 cores
- Disk: Minimum 20GB
- Network: Connectivity between nodes
- User with sudo privileges or root access

## 📁 Project Structure

```
k8s-ansible/
├── ansible.cfg              # Ansible configuration
├── inventory                # Inventory file (list of hosts)
├── site.yml                 # Main playbook
├── group_vars/
│   └── all.yml             # Global configuration variables
├── roles/
│   ├── common/             # Prerequisites and system setup
│   ├── container-runtime/  # Containerd installation
│   ├── kubernetes/         # Kubeadm, kubelet, kubectl
│   ├── master/             # Control plane initialization
│   ├── worker/             # Worker node join
│   ├── cni/                # CNI plugin installation
│   └── addons/             # Optional addons
└── README.md
```

## ⚙️ Configuration

### 1. Edit Inventory File

Edit the `inventory` file and adjust with your node IP addresses:

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

### 2. Variable Configuration

Edit `group_vars/all.yml` to customize the deployment:

#### Kubernetes Version
```yaml
kubernetes_version: "1.34.0"
kubernetes_version_rhel_package: "1.34.0-0"
```

#### CNI Plugin
Choose one: `calico`, `flannel`, `weave`, or `cilium`
```yaml
cni_plugin: "calico"
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

## 🎯 How to use

### 1. Test connectivity to all nodes

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
ansible-playbook site.yml
```

### 3. Verify Deployment

After deployment finishes, log in to the master node and run:

```bash
kubectl get nodes
kubectl get pods -A
```

### 4. Deploy Only Specific Addons

If the cluster is already running and you only want to add addons:

```bash
ansible-playbook site.yml --tags addons
```

## 🔧 Advanced Deployment Options

### Deploy with Specific Tags

```bash
# Only setup prerequisites
ansible-playbook site.yml --tags common

# Only install container runtime
ansible-playbook site.yml --tags container-runtime

# Only install Kubernetes packages
ansible-playbook site.yml --tags kubernetes
```

### Limit Execution to Specific Hosts

```bash
# Only run on master
ansible-playbook site.yml --limit master

# Only run on a specific worker
ansible-playbook site.yml --limit k8s-worker1
```

### Dry Run (Check Mode)

```bash
ansible-playbook site.yml --check
```

## 📝 Configuration Examples

### Example 1: Production Setup with Calico

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

### Example 2: Development Setup with Flannel

```yaml
kubernetes_version: "1.27.0"
cni_plugin: "flannel"
pod_network_cidr: "10.244.0.0/16"

addons:
  metallb:
    enabled: false
  openebs:
    enabled: false
  metrics_server:
    enabled: true
```

### Example 3: Setup with Private Registry

```yaml
kubernetes_version: "1.28.0"
cni_plugin: "calico"
```

## 🔍 Troubleshooting

### Cluster Initialization Failed

If initialization fails, reset the cluster on the master node:

```bash
kubeadm reset -f
rm -rf /etc/kubernetes/
rm -rf /var/lib/etcd/
```

Then rerun the playbook.

### Worker Node Cannot Join

1. Ensure the firewall allows required connections
2. Check if the join token is still valid:
   ```bash
   kubeadm token list
   ```
3. Generate a new token if needed:
   ```bash
   kubeadm token create --print-join-command
   ```

### CNI Plugin Not Working

Check the status of CNI pods:
```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system <cni-pod-name>
```

### Addons Not Installed

Ensure addons are enabled in `group_vars/all.yml` and run:
```bash
ansible-playbook site.yml --tags addons
```

## 🛡️ Security Considerations

1. **SSH Keys**: Use SSH key authentication instead of passwords
2. **Firewall**: Configure firewall to restrict access
3. **RBAC**: Implement RBAC policies after the cluster is running
4. **Network Policies**: Use network policies for pod isolation
5. **Registry Security**: Use TLS for container registries in production

## 📚 References

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Ansible Documentation](https://docs.ansible.com/)
- [Calico Documentation](https://docs.projectcalico.org/)
- [Flannel Documentation](https://github.com/flannel-io/flannel)
- [MetalLB Documentation](https://metallb.universe.tf/)
- [OpenEBS Documentation](https://openebs.io/docs)

## 🤝 Contributing

Contributions are very welcome! Please open a pull request or an issue for improvements.

## ✨ Author

Created with ❤️ for Kubernetes automation