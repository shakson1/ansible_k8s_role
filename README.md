# Kubernetes On-Premise Deployment with Ansible

[![Ansible](https://img.shields.io/badge/Ansible-2.9+-blue.svg)](https://www.ansible.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28+-326CE5.svg?logo=kubernetes)](https://kubernetes.io/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

This Ansible role deploys a highly available Kubernetes cluster on-premise with separated API servers, load balancers, MetalLB, and Istio ingress controller.

## Table of Contents

- [Architecture](#🏗️-architecture)
- [Prerequisites](#✅-prerequisites)
- [Quick Start](#🚀-quick-start)
- [Installation](#📦-installation)
- [Configuration](#⚙️-configuration)
- [Post-Deployment](#🔍-post-deployment)
- [Troubleshooting](#🔧-troubleshooting)
- [Network Requirements](#🌐-network-requirements)
- [Project Structure](#📁-project-structure)
- [Key Features](#🎯-key-features)
- [Notes](#notes)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

## 🏗️ Architecture

This deployment creates a highly available Kubernetes cluster with the following components:

### Node Structure

- **2 API Server Load Balancer Nodes** (`k8s-api-01`, `k8s-api-02`)
  - Running HAProxy for API server load balancing
  - Keepalived for virtual IP (VIP) management
  - Virtual IP: `k8s-api` (configurable)

- **5 Master Nodes** (`k8s-master-01` through `k8s-master-05`)
  - Kubernetes control plane components
  - High availability with multiple masters
  - Etcd running locally on each master

- **10 Worker Nodes** (`k8s-worker-01` through `k8s-worker-10`)
  - Kubernetes worker nodes for running workloads

### Components

- **MetalLB**: Network load balancer for LoadBalancer services
- **Istio**: Service mesh and ingress controller
- **Calico CNI**: Pod networking and network policies

**Total: 17 nodes** (2 API servers + 5 masters + 10 workers)

## ✅ Prerequisites

### System Requirements

All nodes must have:
- **OS**: Ubuntu 20.04/22.04 or RHEL/CentOS 7/8/9
- **Resources**: Minimum 2 CPU cores and 4GB RAM per node (more recommended)
- **Access**: Root or sudo access
- **Network**: Connectivity between all nodes on the same network segment

### Ansible Control Machine

- **Ansible**: Version 2.9 or later
- **Python**: Python 3
- **SSH**: Access to all target nodes

### Network Requirements

- All nodes must be on the same network segment
- API server VIP must be on the same subnet as nodes
- MetalLB IP pool must be on the same subnet
- Required ports must be open (see [Network Requirements](#network-requirements))
- Network interface name must be identified (for keepalived - check with `ip addr`)

## 🚀 Quick Start

### Prerequisites Check

Before running the playbook, ensure:

1. **All nodes are accessible via SSH:**
   ```bash
   ansible all -i inventory -m ping
   ```

2. **Network configuration is verified:**
   - All nodes are on the same network segment
   - API server VIP is configured in `group_vars/all.yml`
   - MetalLB IP pool doesn't conflict with existing IPs
   - Network interface name for keepalived is correct (check with `ip addr`)

### Deployment Steps

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd ansible_kubernetes
   ```

2. **Copy and edit inventory:**
   ```bash
   cp inventory.example inventory
   # Edit inventory with your node IPs
   ```

3. **Configure variables:**
   ```bash
   # Edit group_vars/all.yml with your network settings
   vi group_vars/all.yml
   ```

4. **Run the playbook:**
   ```bash
   ansible-playbook -i inventory playbook.yml
   ```

## 📦 Installation

### Step-by-Step Guide

1. **Prepare inventory file:**
   ```bash
   cp inventory.example inventory
   ```

2. **Edit inventory file:**
   Update the IP addresses and hostnames according to your environment:
   ```ini
   [api-server]
   k8s-api-01 ansible_host=192.168.1.11
   k8s-api-02 ansible_host=192.168.1.12
   
   [master]
   k8s-master-01 ansible_host=192.168.1.21
   k8s-master-02 ansible_host=192.168.1.22
   k8s-master-03 ansible_host=192.168.1.23
   k8s-master-04 ansible_host=192.168.1.24
   k8s-master-05 ansible_host=192.168.1.25
   
   [worker]
   k8s-worker-01 ansible_host=192.168.1.31
   k8s-worker-02 ansible_host=192.168.1.32
   # ... (continue for all 10 workers)
   ```

3. **Configure variables:**
   Edit `group_vars/all.yml` to set:
   - `api_server_vip`: Virtual IP for API server (e.g., "192.168.1.100")
   - `metallb_address_pool`: IP range for MetalLB (e.g., "192.168.1.200-192.168.1.250")
   - `pod_network_cidr`: Pod network CIDR (default: "10.244.0.0/16")
   - `service_cidr`: Service network CIDR (default: "10.96.0.0/12")
   - `keepalived_interface`: Network interface name (e.g., "eth0", "ens33")

4. **Test connectivity:**
   ```bash
   ansible all -i inventory -m ping
   ```

5. **Deploy the cluster:**
   ```bash
   ansible-playbook -i inventory playbook.yml
   ```

### Deployment Order

The playbook automatically handles deployment in the correct order:

1. **Prerequisites** (swap, SELinux, kernel modules, IP forwarding)
2. **Container runtime** (containerd installation and configuration)
3. **Kubernetes components** (kubeadm, kubelet, kubectl)
4. **API server load balancers** (HAProxy + Keepalived on API server nodes)
5. **First master node initialization** (cluster bootstrap)
6. **Additional master nodes join** (high availability setup)
7. **Worker nodes join** (scaling the cluster)
8. **MetalLB installation** (LoadBalancer services)
9. **Istio installation** (Service mesh and ingress)

## ⚙️ Configuration

### Default Variables

Key variables (can be overridden in `group_vars/all.yml`):

| Variable | Default | Description |
|----------|---------|-------------|
| `k8s_version` | `"1.28.0"` | Kubernetes version |
| `api_server_vip` | `"192.168.1.100"` | Virtual IP for API server |
| `api_server_port` | `6443` | API server port |
| `metallb_version` | `"v0.13.12"` | MetalLB version |
| `metallb_address_pool` | `"192.168.1.200-192.168.1.250"` | IP range for MetalLB |
| `pod_network_cidr` | `"10.244.0.0/16"` | Pod network CIDR |
| `service_cidr` | `"10.96.0.0/12"` | Service network CIDR |
| `keepalived_interface` | `"eth0"` | Network interface for keepalived |
| `keepalived_vip_netmask` | `24` | Netmask for VIP (CIDR notation) |
| `keepalived_priority` | `100` | Keepalived priority (first API server) |
| `keepalived_virtual_router_id` | `51` | Keepalived virtual router ID |
| `calico_version` | `"v3.26.1"` | Calico CNI version |
| `istio_version` | `"1.19.0"` | Istio version |
| `firewall_type` | `"auto"` | Firewall type: auto, firewalld, ufw, or none |
| `skip_validation` | `false` | Skip pre-flight validation checks |

See `roles/k8s-onprem/defaults/main.yml` for all available variables.

### Example Configuration

```yaml
# group_vars/all.yml
k8s_version: "1.28.0"
api_server_vip: "192.168.1.100"
metallb_address_pool: "192.168.1.200-192.168.1.250"
keepalived_interface: "eth0"
```

## 🔍 Post-Deployment

### Access the Cluster

1. **SSH to the first master node:**
   ```bash
   ssh k8s-master-01
   kubectl get nodes
   ```

2. **Verify all nodes are ready:**
   ```bash
   kubectl get nodes -o wide
   ```

### Verification Commands

**Check system components:**
```bash
# Check all system pods
kubectl get pods -n kube-system

# Check Calico CNI
kubectl get pods -n kube-system | grep calico
```

**Verify MetalLB:**
```bash
# Check MetalLB pods
kubectl get pods -n metallb-system

# Check IP address pool
kubectl get ipaddresspool -n metallb-system

# Check L2Advertisement
kubectl get l2advertisement -n metallb-system
```

**Verify Istio:**
```bash
# Check Istio pods
kubectl get pods -n istio-system

# Check Istio services
kubectl get svc -n istio-system

# Check Istio version
istioctl version
```

**Verify API server load balancing:**
```bash
# Check Keepalived status on API server nodes
ssh k8s-api-01 systemctl status keepalived
ssh k8s-api-02 systemctl status keepalived

# Check VIP assignment
ssh k8s-api-01 ip addr show | grep <api_server_vip>

# Check HAProxy status
ssh k8s-api-01 systemctl status haproxy
```

## 🔧 Troubleshooting

### Master Nodes Not Joining

**Symptoms:** Additional master nodes fail to join the cluster

**Solutions:**
- Check network connectivity to API server VIP:
  ```bash
  curl -k https://<api-server-vip>:6443
  ```
- Verify keepalived is running on API server nodes:
  ```bash
  ssh k8s-api-01 systemctl status keepalived
  ssh k8s-api-02 systemctl status keepalived
  ```
- Check VIP is assigned:
  ```bash
  ssh k8s-api-01 ip addr show
  ```
- Verify HAProxy is running and configured:
  ```bash
  ssh k8s-api-01 systemctl status haproxy
  ssh k8s-api-01 cat /etc/haproxy/haproxy.cfg
  ```
- Check firewall rules are open on all nodes

### Worker Nodes Not Joining

**Symptoms:** Worker nodes fail to join the cluster

**Solutions:**
- Ensure cluster is fully initialized (all masters ready):
  ```bash
  kubectl get nodes
  ```
- Check token validity (on first master):
  ```bash
  ssh k8s-master-01 kubeadm token list
  ```
- Verify network connectivity to API server VIP:
  ```bash
  curl -k https://<api-server-vip>:6443
  ```
- Check firewall rules allow required ports

### MetalLB Not Working

**Symptoms:** LoadBalancer services don't get IP addresses

**Solutions:**
- Check MetalLB pods are running:
  ```bash
  kubectl get pods -n metallb-system
  ```
- Verify IP pool is configured:
  ```bash
  kubectl get ipaddresspool -n metallb-system
  kubectl describe ipaddresspool default-pool -n metallb-system
  ```
- Check L2Advertisement is configured:
  ```bash
  kubectl get l2advertisement -n metallb-system
  ```
- Ensure IP pool range is on the same network segment as nodes
- Check MetalLB logs:
  ```bash
  kubectl logs -n metallb-system -l app=metallb
  ```

### Istio Issues

**Symptoms:** Istio pods not running or ingress not working

**Solutions:**
- Check Istio pods status:
  ```bash
  kubectl get pods -n istio-system
  ```
- Verify ingress gateway service:
  ```bash
  kubectl get svc -n istio-system
  kubectl describe svc istio-ingressgateway -n istio-system
  ```
- Check Istio logs:
  ```bash
  kubectl logs -n istio-system -l app=istiod
  kubectl logs -n istio-system -l app=istio-ingressgateway
  ```
- Verify Istio installation:
  ```bash
  istioctl verify-install
  ```

### General Issues

**Check cluster status:**
```bash
kubectl cluster-info
kubectl get componentstatuses
```

**Check system resources:**
```bash
kubectl top nodes
kubectl top pods -n kube-system
```

**Review Ansible playbook output:**
```bash
ansible-playbook -i inventory playbook.yml -vvv
```

## 🌐 Network Requirements

### Required Ports

The following ports must be open on all nodes:

| Port | Protocol | Component | Description |
|------|----------|-----------|-------------|
| 6443 | TCP | API Server | Kubernetes API server |
| 2379 | TCP | etcd | etcd client API |
| 2380 | TCP | etcd | etcd peer communication |
| 10250 | TCP | Kubelet | Kubelet API |
| 10259 | TCP | kube-scheduler | kube-scheduler |
| 10257 | TCP | kube-controller-manager | kube-controller-manager |
| 30000-32767 | TCP | NodePort | NodePort service range |

### Firewall Configuration

The playbook automatically configures firewall rules using firewalld. If you're using a different firewall solution, ensure these ports are open manually.

## 📁 Project Structure

```
ansible_kubernetes/
├── ansible.cfg                 # Ansible configuration
├── inventory.example            # Example inventory file
├── playbook.yml                # Main playbook
├── group_vars/
│   └── all.yml                 # Group variables
├── roles/
│   └── k8s-onprem/
│       ├── defaults/
│       │   └── main.yml        # Default variables
│       ├── handlers/
│       │   └── main.yml        # Service handlers
│       ├── tasks/
│       │   ├── main.yml        # Main task file
│       │   ├── api-server.yml  # API server tasks
│       │   ├── containerd.yml  # Container runtime tasks
│       │   ├── istio.yml       # Istio installation
│       │   ├── keepalived.yml  # Keepalived configuration
│       │   ├── kubernetes.yml  # Kubernetes installation
│       │   ├── master.yml      # Master node tasks
│       │   ├── metallb.yml     # MetalLB installation
│       │   ├── prerequisites.yml # System prerequisites
│       │   └── worker.yml      # Worker node tasks
│       └── templates/
│           ├── haproxy.cfg.j2   # HAProxy configuration
│           ├── istio-ingress-gateway.yaml.j2 # Istio gateway
│           ├── keepalived.conf.j2 # Keepalived config
│           ├── kubeadm-init-config.yaml.j2 # Kubeadm config
│           └── metallb-ip-pool.yaml.j2 # MetalLB IP pool
└── README.md                    # This file
```

## 🎯 Key Features

### Pre-flight Checks
- Validates inventory structure and required groups
- Checks system requirements (CPU, RAM, OS)
- Verifies required variables are set
- Warns if API server VIP is already in use

### High Availability
- Keepalived for API server VIP with automatic failover
- HAProxy for API server load balancing with health checks
- Multiple master nodes for control plane redundancy

### Firewall Support
- Automatic detection (firewalld/ufw)
- Manual selection via `firewall_type` variable
- Configures all required ports automatically

### Error Handling
- Validates join commands before execution
- Handles "already a member" scenarios gracefully
- Better error messages for troubleshooting

### Tags for Selective Execution
You can run specific parts of the playbook using tags:

```bash
# Run only prerequisites
ansible-playbook -i inventory playbook.yml --tags prerequisites

# Run only API server configuration
ansible-playbook -i inventory playbook.yml --tags api-server

# Run only firewall configuration
ansible-playbook -i inventory playbook.yml --tags firewall

# Skip validation
ansible-playbook -i inventory playbook.yml -e skip_validation=true
```

Available tags:
- `prerequisites` - System prerequisites
- `api-server` - API server nodes
- `master` - Master nodes
- `worker` - Worker nodes
- `firewall` - Firewall configuration
- `preflight` - Pre-flight checks
- `validation` - Validation tasks

## Notes

- The first master node (`k8s-master-01`) is responsible for initializing the cluster and installing CNI, MetalLB, and Istio
- API server nodes run Keepalived in master/backup mode with automatic failover
- HAProxy includes TCP health checks for better reliability
- All certificates include the API server VIP and all node hostnames/IPs in SANs
- The deployment uses containerd as the container runtime
- Calico is used as the CNI provider (configurable)
- Sysctl settings are persisted in `/etc/sysctl.d/99-kubernetes.conf`
- Kernel modules (br_netfilter, overlay) are checked before loading

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is provided as-is for deployment purposes.

## Acknowledgments

- [Kubernetes](https://kubernetes.io/)
- [Ansible](https://www.ansible.com/)
- [MetalLB](https://metallb.universe.tf/)
- [Istio](https://istio.io/)
- [Calico](https://www.tigera.io/project-calico/)
