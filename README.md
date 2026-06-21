# Kubernetes Cluster Deployment

Production-grade Kubernetes cluster deployment using kubeadm with high-availability control plane, auto-scaling, monitoring, and disaster recovery across multiple availability zones.

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                   │
│                                                        │
│  ┌────────────────────────────────────────────────┐   │
│  │           Control Plane (x3 HA)                │   │
│  │  ┌─────────┐ ┌──────────┐ ┌────────────────┐  │   │
│  │  │  API    │ │Scheduler │ │Controller Mgr  │  │   │
│  │  │ Server  │ │          │ │                │  │   │
│  │  └────┬────┘ └──────────┘ └────────────────┘  │   │
│  │       │                                         │   │
│  │  ┌────▼────┐                                    │   │
│  │  │  etcd   │                                    │   │
│  │  └─────────┘                                    │   │
│  └────────────────────────────────────────────────┘   │
│              │                    │                    │
│  ┌───────────┴──────────┐ ┌──────┴───────────────┐   │
│  │   Worker Node 1      │ │   Worker Node N      │   │
│  │  ┌────────────────┐  │ │  ┌────────────────┐  │   │
│  │  │ Kubelet │Proxy  │  │ │  │ Kubelet │Proxy  │  │   │
│  │  │  Pods   │Calico │  │ │  │  Pods   │Calico │  │   │
│  │  └────────────────┘  │ │  └────────────────┘  │   │
│  └──────────────────────┘ └──────────────────────┘   │
│                                                        │
│  ┌────────────────────────────────────────────────┐   │
│  │           Additional Components                │   │
│  │  ┌──────────┐ ┌──────────┐ ┌────────────────┐  │   │
│  │  │  MetalLB │ │ Ingress  │ │   Rook/Ceph    │  │   │
│  │  │  LB      │ │ Nginx    │ │   Storage      │  │   │
│  │  └──────────┘ └──────────┘ └────────────────┘  │   │
│  │  ┌──────────┐ ┌──────────┐ ┌────────────────┐  │   │
│  │  │ Prometh. │ │ Grafana  │ │   Metrics      │  │   │
│  │  │ Monitor  │ │ Dashboard│ │   Server       │  │   │
│  │  └──────────┘ └──────────┘ └────────────────┘  │   │
│  └────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

## Features

- **High-Availability Control Plane**: 3 master nodes with stacked etcd
- **Auto-Scaling**: Cluster Autoscaler + Horizontal Pod Autoscaler
- **Network**: Calico CNI with Network Policies
- **Load Balancing**: MetalLB for bare-metal load balancing
- **Storage**: Rook/Ceph for persistent storage
- **Monitoring**: Prometheus + Grafana stack
- **Ingress**: NGINX Ingress Controller
- **RBAC**: Role-based access control with least privilege

## Prerequisites

- 6+ servers (3 master, 3+ worker) with Ubuntu 22.04/RHEL 9
- 4GB RAM, 2 CPU per node minimum
- Network connectivity between all nodes
- Root/sudo access

## Installation

### 1. Install Container Runtime

```bash
# Install containerd on all nodes
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

### 2. Install kubeadm, kubelet, kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 3. Initialize Control Plane

```bash
# On first master node
sudo kubeadm init --control-plane-endpoint "cluster-lb.example.com:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# Set up kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4. Install CNI (Calico)

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26/manifests/custom-resources.yaml
```

### 5. Join Worker Nodes

```bash
# On each worker node
sudo kubeadm join cluster-lb.example.com:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

## Usage

### Deploy an Application

```bash
kubectl create deployment nginx --image=nginx:alpine
kubectl expose deployment nginx --port=80 --type=LoadBalancer
kubectl get pods,svc
```

### Scale Workloads

```bash
kubectl scale deployment nginx --replicas=5
kubectl autoscale deployment nginx --min=3 --max=10 --cpu-percent=80
```

## Commands

| Command | Description |
|---------|-------------|
| `kubectl get nodes` | List all cluster nodes |
| `kubectl get pods -A` | List all pods across namespaces |
| `kubectl describe node <name>` | Detailed node information |
| `kubectl top nodes` | Node resource usage |
| `kubectl cordon <node>` | Mark node as unschedulable |
| `kubectl drain <node>` | Evict pods from node for maintenance |
| `kubectl uncordon <node>` | Mark node as schedulable again |
| `kubectl cluster-info` | Display cluster information |

## Best Practices

- Use namespaces for resource isolation
- Set resource requests/limits on all containers
- Enable audit logging at the API server level
- Regular etcd backups to S3-compatible storage
- Use Pod Security Standards (baseline/restricted)
- Implement network policies for micro-segmentation
- Regular cluster upgrades (one minor version at a time)

## Security Considerations

- Enable RBAC and use least-privilege service accounts
- Restrict API server access with firewall rules
- Encrypt etcd data at rest
- Scan container images for vulnerabilities
- Use OPA/Gatekeeper for policy enforcement
- Enable Kubernetes Dashboard with RBAC only
- Rotate certificates before expiration

## Monitoring Setup

```bash
# Install Prometheus stack
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
```
