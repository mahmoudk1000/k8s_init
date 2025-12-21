# Kubernetes Cluster Automation with Ansible

[![CI](https://github.com/YOUR_USERNAME/kubernetes-init/actions/workflows/ci.yml/badge.svg)](https://github.com/YOUR_USERNAME/kubernetes-init/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Ansible](https://img.shields.io/badge/ansible-%3E%3D2.15-blue.svg)](https://docs.ansible.com/)

This Ansible playbook automates the deployment and configuration of Kubernetes clusters with support for multiple control planes, workers, and flexible configuration options.

## Features

### 1. **Variable Kubernetes Version**
You can now specify the Kubernetes version you want to install by setting the `kube_version` variable in `group_vars/all.yml`:

```yaml
kube_version: "1.30"  # Format: "1.30", "1.29", "1.28", etc.
```

### 2. **Variable Runtime Engine Version**
Configure the container runtime version through variables:

```yaml
containerd_version: "latest"  # Use "latest" or specific version like "1.7.13-1"
crio_version: "1.30"          # CRI-O version (should match Kubernetes major.minor)
```

### 3. **Support for Multiple Container Runtimes**
Choose between containerd and CRI-O as your container runtime:

```yaml
container_runtime: "containerd"  # Options: containerd, crio
```

The playbook will automatically install and configure the selected runtime.

### 4. **Multiple Control Planes (HA Setup)**
Deploy highly available Kubernetes clusters with multiple control plane nodes:

1. Define multiple control plane nodes in your inventory:
```ini
[control_plane]
master1 ansible_host=192.168.1.10 node_name=k8s-master-01
master2 ansible_host=192.168.1.11 node_name=k8s-master-02
master3 ansible_host=192.168.1.12 node_name=k8s-master-03
```

2. Set the control plane endpoint in `group_vars/all.yml`:
```yaml
control_plane_endpoint: "192.168.1.100:6443"  # Load balancer or VIP
```

### 5. **Multiple Workers**
Add as many worker nodes as needed in your inventory:

```ini
[workers]
worker1 ansible_host=192.168.1.20 node_name=k8s-worker-01
worker2 ansible_host=192.168.1.21 node_name=k8s-worker-02
worker3 ansible_host=192.168.1.22 node_name=k8s-worker-03
```

You can add new workers to an existing cluster by:
1. Adding them to the inventory file
2. Running the playbook again (it will skip already configured nodes)

### 6. **Custom Node Names**
Set custom Kubernetes node names for each host in the inventory:

```ini
master ansible_host=108.142.236.82 node_name=k8s-master-01
worker1 ansible_host=108.143.75.163 node_name=k8s-worker-01
```

If `node_name` is not specified, the inventory hostname will be used.

### 7. **Existing User Support**
The playbook now uses an existing user instead of creating a new one. Set your user in the inventory or group_vars:

```yaml
kube_user: '{{ ansible_user }}'  # Uses the ansible_user for kubectl operations
```

The playbook will configure sudo access for this user without creating a new account.

### 8. **Node Labeling**
Apply custom labels to your nodes for workload scheduling and organization:

**In inventory file:**
```ini
worker1 ansible_host=108.143.75.163 node_name=k8s-worker-01 node_labels='{"environment":"production","workload":"backend","zone":"us-east"}'
```

**Or globally in group_vars/all.yml:**
```yaml
node_labels:
  environment: production
  role: application
```

Labels can be used for:
- Node affinity and anti-affinity
- Workload placement
- Resource organization
- Monitoring and management

### 9. **Multiple CNI Options**
Choose your preferred Container Network Interface (CNI) plugin or skip CNI installation entirely. All CNI plugins are installed using Helm for easy management:

```yaml
install_cni: true              # Set to false to skip CNI installation
cni_plugin: "flannel"          # Options: flannel, calico, cilium
```

**Supported CNI Plugins (via Helm):**
- **Flannel**: Simple overlay network, great for basic setups
- **Calico**: Advanced networking with network policies
- **Cilium**: eBPF-based networking with advanced security features

CNI-specific Helm chart versions can be configured:
```yaml
calico_version: "v3.27.0"      # Tigera operator Helm chart version
cilium_version: "1.14.5"       # Cilium Helm chart version
```

### 10. **Optional ArgoCD Installation**
Automatically install ArgoCD for GitOps workflows using the official Helm chart:

```yaml
install_argocd: true           # Set to true to install ArgoCD
argocd_version: "5.51.6"       # Helm chart version (latest stable)
argocd_namespace: "argocd"
```

When installed, ArgoCD credentials are saved to `~/argocd-credentials.txt` on the master node.

**Manage ArgoCD with Helm:**
```bash
# List Helm releases
helm list -n argocd

# Upgrade ArgoCD
helm upgrade argocd argo/argo-cd -n argocd

# Uninstall ArgoCD
helm uninstall argocd -n argocd
```

## Quick Start

### Prerequisites
- Ansible installed on your control machine
- SSH access to all target nodes
- Target nodes running Ubuntu/Debian
- Python 3 on all target nodes

### Installation

1. Clone this repository:
```bash
git clone <repository-url>
cd kubernetes-init
```

2. Edit the inventory file (`hosts.ini`):
```ini
[control_plane]
master1 ansible_host=YOUR_MASTER_IP node_name=k8s-master-01

[workers]
worker1 ansible_host=YOUR_WORKER1_IP node_name=k8s-worker-01
worker2 ansible_host=YOUR_WORKER2_IP node_name=k8s-worker-02

[cluster:vars]
ansible_user=YOUR_SSH_USER
ansible_ssh_private_key_file=~/.ssh/YOUR_KEY
```

3. Configure your cluster in `group_vars/all.yml`:
```yaml
kube_version: "1.30"
container_runtime: "containerd"
containerd_version: "latest"
pod_network_cidr: 10.244.0.0/16
cni_plugin: "flannel"
install_argocd: false
```

4. Run the playbook:
```bash
ansible-playbook -i hosts.ini main.yml
```

## Advanced Configuration

### CNI Selection

Choose your preferred CNI plugin:

**Flannel (Default - Simple and Reliable):**
```yaml
install_cni: true
cni_plugin: "flannel"
```

**Calico (Network Policies):**
```yaml
install_cni: true
cni_plugin: "calico"
calico_version: "v3.27.0"
```

**Cilium (eBPF-based):**
```yaml
install_cni: true
cni_plugin: "cilium"
cilium_version: "1.14.5"
```

**No CNI (Bring Your Own):**
```yaml
install_cni: false
```

### ArgoCD Installation

Enable ArgoCD for GitOps:
```yaml
install_argocd: true
argocd_version: "stable"
argocd_namespace: "argocd"
```

After installation, access ArgoCD:
```bash
# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get credentials from master node
cat ~/argocd-credentials.txt
```

### High Availability Setup

For HA control plane:

1. Set up a load balancer (HAProxy, Nginx, or cloud LB) in front of your control plane nodes
2. Configure the endpoint in `group_vars/all.yml`:
```yaml
control_plane_endpoint: "lb.example.com:6443"
```

3. Define all control plane nodes in inventory
4. Run the playbook

### Adding Nodes to Existing Cluster

To add new nodes to an existing cluster:

1. Add the new nodes to your inventory file
2. Re-run the playbook:
```bash
ansible-playbook -i hosts.ini main.yml
```

The playbook will skip already configured nodes and only join the new ones.

### Using CRI-O Instead of Containerd

Edit `group_vars/all.yml`:
```yaml
container_runtime: "crio"
crio_version: "1.30"  # Should match your Kubernetes version
```

## File Structure

```
.
├── main.yml                    # Main playbook
├── hosts.ini                   # Inventory file
├── group_vars/
│   └── all.yml                # Global variables
└── roles/
    ├── foundation/            # Base system setup and runtime installation
    ├── init_master/           # Control plane initialization
    ├── post_master/           # Post-init master configuration
    ├── init_node/             # Node joining (workers and additional masters)
    └── post_node/             # Post-join node configuration
```

## Troubleshooting

### Checking Cluster Status
```bash
kubectl get nodes
kubectl get pods -A
```

### Viewing Node Labels
```bash
kubectl get nodes --show-labels
```

### Re-joining a Node
If a node fails to join:
1. Remove the join log file on the node: `rm ~/node_joined.log` or `~/control_plane_joined.log`
2. Re-run the playbook

### Changing Container Runtime
To switch container runtimes, you'll need to:
1. Update the `container_runtime` variable
2. Drain and reset nodes
3. Re-run the playbook

## Testing and CI/CD

### Local Testing

Use the provided `Makefile` for local testing:

```bash
# Show all available commands
make help

# Run linting checks
make lint

# Check playbook syntax
make syntax-check

# Validate inventory and roles
make validate

# Run all tests
make test

# Dry run without making changes
make dry-run

# Run everything including security scan
make all
```

### Pre-commit Hooks

Install pre-commit hooks to catch issues before committing:

```bash
pip install pre-commit
pre-commit install

# Run manually on all files
pre-commit run --all-files
```

### CI/CD Pipeline

The project includes a comprehensive GitHub Actions workflow that automatically runs on:
- Pushes to main/master/develop branches
- Pull requests
- Manual workflow dispatch

**Pipeline Jobs:**
1. **Lint** - yamllint and ansible-lint checks
2. **Syntax Check** - Validates playbook and inventory syntax
3. **Validate Roles** - Tests each role individually
4. **Security Scan** - Trivy vulnerability scanning
5. **Documentation Check** - Validates README and documentation links
6. **Test Inventory** - Validates inventory structure
7. **Test Variables** - Checks required variables exist
8. **Test Dry Run** - Runs playbook in check mode
9. **Test Role Dependencies** - Validates role structure

**View pipeline results:**
- Go to the "Actions" tab in your GitHub repository
- Security scan results appear in the "Security" tab

For detailed testing information, see [TESTING.md](TESTING.md).

## Variables Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `kube_version` | "1.30" | Kubernetes version to install |
| `container_runtime` | "containerd" | Container runtime (containerd or crio) |
| `containerd_version` | "latest" | Containerd version |
| `crio_version` | "1.30" | CRI-O version |
| `pod_network_cidr` | "10.244.0.0/16" | Pod network CIDR |
| `control_plane_endpoint` | "" | HA control plane endpoint |
| `install_cni` | true | Whether to install CNI plugin via Helm |
| `cni_plugin` | "flannel" | CNI plugin (flannel, calico, cilium) |
| `calico_version` | "v3.27.0" | Calico Helm chart version |
| `cilium_version` | "1.14.5" | Cilium Helm chart version |
| `install_argocd` | false | Whether to install ArgoCD via Helm |
| `argocd_version` | "5.51.6" | ArgoCD Helm chart version |
| `argocd_namespace` | "argocd" | ArgoCD namespace |
| `kube_user` | "{{ ansible_user }}" | User for kubectl operations |
| `node_labels` | {} | Default node labels |

## Documentation

- [QUICKSTART.md](QUICKSTART.md) - Quick start guide with examples
- [CNI_GUIDE.md](CNI_GUIDE.md) - CNI plugin comparison and management
- [ARGOCD_GUIDE.md](ARGOCD_GUIDE.md) - ArgoCD installation and usage
- [MIGRATION.md](MIGRATION.md) - Migration guide from v1.x
- [TESTING.md](TESTING.md) - Testing guide and CI/CD information
- [IMPLEMENTATION_SUMMARY.md](IMPLEMENTATION_SUMMARY.md) - Feature implementation summary

## Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Run tests locally with `make test`
4. Submit a pull request

All pull requests must pass CI/CD checks before merging.

## License

MIT

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.
